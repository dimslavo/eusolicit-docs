# AgenticSAI Platform — API Contract & Integration Guide

> **Audience:** engineers and autonomous AI agents designing third-party integrations and building architecture on top of the AgenticSAI platform.
>
> **Source of truth:** OpenAPI `3.1.0` spec `AgenticSAI API` v1.0 (`api-docs_AgenticSAI_.json`). This document is generated from and consistent with that spec.
>
> **Generated:** 2026-05-17 · **Endpoints:** 404 operations across 305 paths · **Schemas:** 364

This guide is self-contained. An integrating system should be able to design and implement a robust client from this document alone: authentication, tenancy model, resource lifecycle, request/response envelopes, pagination, streaming, error handling, webhooks, and the full endpoint + schema reference.

> ### ⚠️ Scope & authority — build to the schema, not to a deployment
>
> This document is the **forward-looking API contract**. It is generated **entirely from the OpenAPI schema** (`api-docs_AgenticSAI_.json`) and intentionally describes the API **as specified**, not as currently running in any particular environment.
>
> - Currently deployed builds (including local docker images) **may lag behind this schema** — some endpoints, fields, or schemas documented here may not yet be live. This is expected; those services are being upgraded to match.
> - **Design and implement against this contract.** Do not narrow an integration to whatever a given environment happens to expose today; treat anything in this guide as the target the platform converges to.
> - When an endpoint here returns `404`/`501` in a not-yet-upgraded environment, treat it as *not-yet-deployed*, not *removed* — feature-flag/retry rather than redesign.
> - If this guide and a live response ever disagree on shape, **the schema (this guide) is authoritative** for integration design; report the drift to the platform operator.

---

## Table of Contents

- [1. Platform Model & Core Concepts](#1-platform-model--core-concepts)
- [2. Base URL, Environments & Versioning](#2-base-url-environments--versioning)
- [3. Authentication & Authorization](#3-authentication--authorization)
- [4. Request/Response Conventions](#4-requestresponse-conventions)
- [5. Pagination, Filtering & Sorting](#5-pagination-filtering--sorting)
- [6. Error Model](#6-error-model)
- [7. Streaming (SSE) Execution](#7-streaming-sse-execution)
- [8. Webhooks (Standard Webhooks spec)](#8-webhooks-standard-webhooks-spec)
- [9. Rate Limiting & Quotas](#9-rate-limiting--quotas)
- [10. Integration Architecture Patterns](#10-integration-architecture-patterns)
- [11. Endpoint Reference (by domain)](#11-endpoint-reference-by-domain)
- [12. Schema Reference](#12-schema-reference)
- [13. Appendix: Full Endpoint Index](#13-appendix-full-endpoint-index)

### Endpoint reference — domains

- [Authentication & Identity](#dom-authentication-identity) — 15 operations
- [Organizations, Projects & Tenancy](#dom-organizations-projects-tenancy) — 50 operations
- [Agents](#dom-agents) — 39 operations
- [Teams](#dom-teams) — 17 operations
- [Workflows](#dom-workflows) — 15 operations
- [Audio, Voice & Telephony](#dom-audio-voice-telephony) — 24 operations
- [Knowledge, Storage & Embeddings](#dom-knowledge-storage-embeddings) — 15 operations
- [Tools & External Integrations](#dom-tools-external-integrations) — 33 operations
- [Observability: Traces, Evaluations & Usage](#dom-observability-traces-evaluations-usage) — 18 operations
- [Governance: Policies & Compliance](#dom-governance-policies-compliance) — 16 operations
- [Webhooks & Eventing](#dom-webhooks-eventing) — 11 operations
- [Billing](#dom-billing) — 14 operations
- [Platform, Config & Media](#dom-platform-config-media) — 97 operations
- [Other](#dom-other) — 40 operations

---

## 1. Platform Model & Core Concepts

AgenticSAI is a multi-tenant platform for building, executing, observing and governing AI
**agents**, **teams** (multi-agent), **workflows**, and **voice/audio agents**, with attached
**knowledge stores**, **tools**, **policies**, and **observability**.

### 1.1 Tenancy hierarchy

```
Organization                      (billing boundary, members, secrets, rate limits, policies)
└── Project                       (isolation unit; owns API keys, resources, runs)
    ├── Agents                    (single LLM agent: model, prompt, tools, memory, knowledge)
    ├── Teams                     (orchestrated groups of agents)
    ├── Workflows                 (graph/sequence of steps; deterministic + AI nodes)
    ├── Audio / Voice agents      (real-time voice via LiveKit/WebRTC + telephony presets)
    ├── Storage Resources         (vector stores / knowledge bases + embeddings)
    ├── HTTP Requests / Skills    (callable tools attached to agents)
    ├── MCP Servers / MCP Tools   (Model Context Protocol tool providers)
    ├── Sessions                  (conversation/run context per agent/team/workflow)
    ├── Runs                      (a single execution + its logs/trace)
    └── Webhook subscriptions     (outbound event delivery)
```

Almost every integration resource is addressed under an organization and/or project. Two URL
families exist (see §3):

| Family | Prefix | Auth | Primary consumer |
|---|---|---|---|
| **Public Integration API (v1)** | `/client/api/v1/...` | `X-API-Key` (project-scoped) | **3rd-party integrations — use this** |
| **Console / Management API** | `/api/...` | `bearer-jwt` (user) / `bearer-token` (system) | First-party UI & admin tooling |

> **Integration rule of thumb:** prefer the `v1` API-key endpoints for machine-to-machine
> integrations. They are stable, project-scoped, and return the uniform `ApiResponse<T>`
> envelope. The `/api/...` console endpoints exist for completeness and admin automation but
> are tied to interactive user sessions and broader privileges.

### 1.2 Key entities

- **Agent** — a configured LLM unit (model/provider, system prompt, tools, memory policy,
  attached knowledge/storage resources). Executed synchronously or via SSE streaming; each
  execution is a **Run** inside a **Session**.
- **Team** — multiple agents under an orchestration strategy; same session/run/trace model.
- **Workflow** — a multi-step graph (AI + deterministic + HTTP/tool nodes); executed with
  streaming or non-streaming; has its own sessions.
- **Session** — durable conversation/run context (history, memory). Identified by
  `session_id`; supports rename, list, delete; optional `dbId`/`user_id` for multi-tenant
  partitioning of session storage.
- **Run** — one execution instance; produces logs, usage, and a **Trace**.
- **Trace / Evaluation** — observability: per-run spans plus offline evaluation runs over
  agents/teams.
- **Storage Resource** — a vector store / knowledge base; documents are chunked + embedded
  (embeddings proxy is OpenAI-compatible).
- **Policy / Compliance Framework** — governance rules enforced on inputs/outputs with audit
  logs and statistics.
- **Webhook subscription** — outbound event delivery following the Standard Webhooks spec.

---

## 2. Base URL, Environments & Versioning

- **Base URL (server):** `https://agenticsai.endigitalx.com`
- **OpenAPI:** `3.1.0` · **API version:** `1.0`
- All paths in this guide are relative to the base URL.
- **Versioning:** the integration surface is versioned in the path (`/client/api/v1/...`).
  Treat any non-`v1` `/api/...` path as internal/console and subject to change. The
  `ApiResponse` envelope also carries a `version` string per response.
- **TLS:** HTTPS only. Send `Content-Type: application/json` for bodies; many responses are
  declared as `*/*` but are JSON in practice.

> Network/IP allow-listing may be enforced in front of the platform; coordinate egress IPs
> with the platform operator before going live.

---

## 3. Authentication & Authorization

The spec declares the following security schemes:

| Scheme | Type | Transport | Purpose |
|---|---|---|---|
| `bearer-jwt` | http / bearer (JWT) | `Authorization: Bearer <jwt>` | Interactive **user** auth for console/management `/api/...` endpoints. Obtained via the Authentication endpoints. |
| `api-key` | apiKey | `X-API-Key: <key>` header | **Project-scoped** authentication for the **Public Integration API** (`/client/api/v1/...`). The primary mechanism for 3rd-party integrations. |
| `service-key-audio` | apiKey | `X-Service-Key: <key>` header | Service-to-service key for **Audio** category endpoints. |
| `service-key-agent-creator` | apiKey | `X-Service-Key: <key>` header | Service-to-service key for **Agent Creator** category endpoints. |
| `service-name` | apiKey | `X-Service-Name: <name>` header | Optional service identity for audit/logging (defaults to `service-<category>`). |
| `bearer-token` | http / bearer | `Authorization: Bearer <token>` | **Not formally declared** in `securitySchemes` but referenced by Webhooks v1 and Documentation Management endpoints. Treat as a system/management bearer token issued by the platform operator. |

### 3.1 Which auth do I use?

| You are building… | Use | Header |
|---|---|---|
| A 3rd-party integration / backend automation | **`api-key`** | `X-API-Key: <project key>` |
| A user-facing console / acting as a logged-in user | `bearer-jwt` | `Authorization: Bearer <jwt>` |
| Webhook subscription management & docs management | `bearer-token` | `Authorization: Bearer <token>` |
| An Audio or Agent-Creator backend service | `service-key-*` (+ `service-name`) | `X-Service-Key`, `X-Service-Name` |

### 3.2 Obtaining credentials

- **Project API keys** (`X-API-Key`) are minted via the *Project API Keys v1* endpoints
  (system-API-key authenticated) and via the console *Organizations/Projects* endpoints.
  A key is bound to one project and carries that project's scope/permissions. Rotate by
  creating a new key and deleting the old one.
- **JWTs** are obtained from the *Authentication* endpoints:
  `POST /api/auth/signup`, `POST /api/auth/login`, `POST /api/auth/refreshAccessToken`,
  `POST /api/auth/google/callback` (Google GSI redirect flow). Tokens are refreshable;
  store the refresh token securely and exchange it before access-token expiry.
- **Role presets / project members** govern fine-grained authorization within an org/project.
  An API key inherits the permission set configured for it; design integrations against the
  least-privilege key.

### 3.3 Auth handling rules for clients

- Always send exactly one credential appropriate to the endpoint family.
- On `401` re-authenticate (refresh JWT, or surface an invalid/rotated API key).
- On `403` the credential is valid but lacks scope — do not retry; escalate.
- Never log full tokens/keys. Treat `X-API-Key` as a project-level secret.

---

## 4. Request/Response Conventions

### 4.1 The `ApiResponse<T>` envelope

The Public Integration API (`/client/api/v1/...`) wraps responses in a uniform envelope:

```json
{
  "success": true,
  "data": { "...": "T — the resource payload" },
  "message": "human-readable status",
  "timestamp": "2026-05-17T12:34:56Z",
  "version": "1.0"
}
```

| Field | Type | Notes |
|---|---|---|
| `success` | boolean | **Always present.** `false` even on some `2xx`-shaped error envelopes — check this, not only HTTP status. |
| `data` | `T` \| null | The typed payload. `null` on errors or for `Unit`/void operations. |
| `message` | string | Human-readable; safe to surface to operators, not end users. |
| `timestamp` | date-time | Server time of response. |
| `version` | string | API/contract version. |

> **Client rule:** treat a response as successful only when HTTP status is `2xx` **and**
> `success == true`. Read the resource from `data`.

Console `/api/...` endpoints may return the resource directly (no envelope) or a Spring
`Page<T>` for collections. Always branch on the documented response schema per endpoint
(§11).

### 4.2 Content types

- Request bodies: `application/json` unless an endpoint documents `multipart/form-data`
  (file upload to storage resources / media).
- Response content is frequently declared `*/*` but is JSON; parse as JSON unless the
  endpoint is a media/image/SSE proxy.

### 4.3 Idempotency & retries

- `GET`/`DELETE`/`PUT` are idempotent — safe to retry with backoff on `5xx`/network errors.
- `POST` that creates resources or triggers runs is **not** inherently idempotent. Use a
  client-side dedupe key (e.g. your own correlation id in the payload/session) and check for
  existing resources before retrying.
- Recommended retry policy: exponential backoff (e.g. 250ms → 4s, max ~5 tries) on
  `429`/`502`/`503`/`504` and connection errors only. Never retry `4xx` (except `429`).

---

## 5. Pagination, Filtering & Sorting

Two pagination shapes appear:

**A. Spring `Page<T>`** (most list endpoints):

```json
{
  "content": [ ...T ],
  "number": 0, "size": 20,
  "totalElements": 137, "totalPages": 7,
  "first": true, "last": false, "numberOfElements": 20, "empty": false,
  "sort": { "...": "SortObject" }, "pageable": { "...": "PageableObject" }
}
```

Common query parameters: `page` (0-based), `size`, `sortBy`, `sortOrder`
(`asc`/`desc`), and a `pageable` object on some endpoints. Iterate until `last == true`
(or `number + 1 >= totalPages`).

**B. `ApiResponse<List...>`** — the envelope wrapping an array in `data` (smaller / bounded
collections). Use `limit` where offered.

**Frequently used filter/scoping query params** (varies per endpoint — see §11):

| Param | Meaning |
|---|---|
| `user_id` | Partition sessions/runs by end-user (multi-tenant within a project). |
| `dbId` | Optional logical DB id for multi-tenant session storage. |
| `session_id` | Target a specific conversation/run context. |
| `agentId` / `teamId` / `workflowId` | Scope to a specific executable. |
| `project_id` / `project_ref_id` / `organization_id` | Tenancy scoping. |
| `startDate` / `endDate` | Time-window filter (traces, usage, audit logs). |
| `tag` / `tagIds` | Tag-based filtering. |
| `scope` | Resource visibility/scope filter. |

---

## 6. Error Model

Status codes observed across the API (count of operations declaring each):

| Status | Count | Meaning & client action |
|---|---|---|
| `200` | 334 | OK — success. |
| `201` | 25 | Created — resource created; read `data`/`Location`. |
| `202` | 3 | Accepted — async accepted (e.g. queued run); poll the run/trace. |
| `204` | 42 | No Content — success, empty body. |
| `400` | 92 | Bad Request — validation error; **do not retry** unchanged. Inspect `message`. |
| `401` | 112 | Unauthorized — missing/invalid/expired credential; re-auth. |
| `403` | 105 | Forbidden — authenticated but lacks scope; escalate, do not retry. |
| `404` | 147 | Not Found — wrong id/tenant or deleted; do not retry. |
| `409` | 14 | Conflict — state/uniqueness conflict; reconcile then retry. |
| `429` | 37 | Too Many Requests — rate limited; back off using `Retry-After`. |
| `500` | 5 | Server Error — retry with backoff. |
| `503` | 55 | Service Unavailable — transient; retry with backoff. |

On the Public API, errors still arrive in the `ApiResponse` envelope with
`success: false`, `data: null`, and a diagnostic `message`. Always branch on
`success`. Console endpoints may return a plain error object — defensively parse
`message`/`error` fields and fall back to the HTTP status.

**Client error-handling checklist**

1. Network/timeout → retry with backoff (idempotent ops only).
2. `2xx` + `success:true` → proceed with `data`.
3. `2xx` + `success:false` → application error; read `message`; no retry.
4. `401` → refresh/re-auth once, then fail.
5. `403`/`404`/`400`/`422` → terminal; report `message`.
6. `429`/`5xx` → backoff + retry budget.

---

## 7. Streaming (SSE) Execution

Agent / Team / Workflow execution supports both **non-streaming** (single JSON response) and **streaming** (Server-Sent Events) variants. For streaming endpoints:

- Send the execution request; read the response as an SSE stream
  (`Accept: text/event-stream`).
- Each event is an incremental token/step/tool-call/trace chunk; concatenate content
  deltas and handle terminal/`done` events.
- Always implement an idle/total timeout and a graceful abort; a dropped stream should
  fall back to fetching the **Run**/**Trace** by id to recover final state.
- Prefer non-streaming for batch/automation; streaming for interactive UX.

Execution / streaming-related operations detected in the spec:

| Method | Path | Summary |
|---|---|---|
| `GET` | `/api/events` |  |
| `GET` | `/api/events/test` |  |
| `GET` | `/api/images/{documentId}/{filename}` | Get image |
| `GET` | `/api/media/{s3KeyEncoded}` | Get media file |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run` | Execute agent (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run/stream` | Execute agent (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/runs/{runId}/continue` | Continue a paused agent run (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/run-stream` |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agents/{agentId}/run-stream` |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/teams/{teamId}/run-stream` |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/workflows/{workflowId}/run-stream` |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run` | Execute team (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run/stream` | Execute team (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run` | Execute workflow (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run/stream` | Execute workflow (streaming) |
| `POST` | `/client/api/v1/agents/{agentId}/run-stream` | Execute agent with streaming response |
| `POST` | `/client/api/v1/agents/{agentId}/runs/{runId}/continue` | Continue an agent run with streaming response |
| `POST` | `/client/api/v1/teams/{teamId}/run` | Execute team (non-streaming) |
| `POST` | `/client/api/v1/teams/{teamId}/run-stream` | Execute team (streaming) |
| `POST` | `/client/api/v1/workflows/{workflowId}/run-stream` | Execute workflow with streaming response |

---

## 8. Webhooks (Standard Webhooks spec)

Outbound events follow the **Standard Webhooks** specification. Manage subscriptions via the
*Webhooks v1* endpoints (auth: `bearer-token`):

- `POST /api/webhooks/subscriptions` — create a subscription (URL + subscribed event types).
- `GET /api/webhooks/subscriptions` / `GET /api/webhooks/subscriptions/{id}` — list / read.
- `PUT /api/webhooks/subscriptions/{id}` / `DELETE` — update / remove.
- `POST /api/webhooks/subscriptions/{id}/test` — send a test delivery.
- `GET /api/webhooks/subscriptions/{subscriptionId}/statistics` — delivery stats.
- `GET /api/webhooks/event-types` — discover supported event types (call this first;
  do not hard-code the catalogue).
- `GET /api/webhooks/deliveries` / `POST /api/webhooks/deliveries/{id}/replay` — inspect &
  replay failed deliveries.

**Receiver requirements**

- Verify the signature header per the Standard Webhooks spec (HMAC over the raw body using
  the subscription's signing secret) **before** trusting the payload.
- Enforce timestamp tolerance to reject replays; deduplicate on the message id.
- Respond `2xx` quickly (ack then process async). Non-`2xx`/timeout → platform retries with
  backoff; use the deliveries/replay endpoints for recovery.
- Treat delivery as at-least-once; make handlers idempotent.

There is also a **Stripe webhook receiver** (`POST /api/webhook/stripe`, public) for billing
events — that is inbound to the platform, not something you subscribe to.

---

## 9. Rate Limiting & Quotas

- Rate limits and quotas are configured **per organization** and surfaced/managed through the
  *Organization Administration* endpoints (e.g. organization limits / rate-limit
  configuration) and reflected in *Usage v1*.
- Expect `429 Too Many Requests` when exceeded; honour `Retry-After` if present, otherwise
  exponential backoff with jitter.
- Track consumption proactively via *Usage v1* (`/client/api/v1/.../usage`) and the
  organization limits endpoints rather than discovering limits by hitting `429`.
- Voice/audio and LLM-execution endpoints are the most quota-sensitive — budget concurrency
  accordingly and prefer non-streaming batch where latency allows.

---

## 10. Integration Architecture Patterns

Concrete recipes for an agent designing a system on top of AgenticSAI.

### 10.1 Standard machine-to-machine integration (recommended)

1. Operator provisions an **Organization** and **Project**; you receive a project
   **`X-API-Key`** (least-privilege via role presets).
2. Discover capabilities: list agents/teams/workflows
   (`GET /client/api/v1/agents`, `/teams`, `/workflows`).
3. For each end-user of *your* product, create/scope a **Session** using a stable
   `user_id` (and optionally `dbId`) so histories/memory stay isolated.
4. Execute: call the agent/team/workflow run endpoint (non-streaming for backends, SSE for
   interactive). Persist the returned `run`/`session` ids.
5. Observe: pull **Traces**/**Runs**/**Usage** for cost, latency, and debugging; wire
   **Webhooks** for async completion instead of polling where possible.
6. Govern: rely on **Policies/Compliance** server-side; surface policy denials from the
   `message`/error to your users.

### 10.2 Knowledge-grounded agent (RAG)

1. Create a **Storage Resource** (vector store) in the project.
2. Upload/ingest documents (multipart) → platform chunks + embeds (OpenAI-compatible
   *Embeddings v1* proxy).
3. Attach the storage resource to an **Agent**.
4. Execute the agent; retrieval is automatic. Use **Traces** to inspect retrieved chunks.

### 10.3 Tool-augmented agent

- Register tools as **HTTP Requests** (declarative outbound calls), **Skills**, or via
  **MCP Servers/Tools** (Model Context Protocol). Attach to the agent. The platform invokes
  them during runs; inspect tool calls in traces.

### 10.4 Voice / telephony integration

- Configure a **Telephony Preset** and **Voice Preset**; create an **Audio Agent**.
- Real-time media is **WebRTC/LiveKit** (SIP URL via `GET /api/public/sip-url`) — media
  does not traverse the REST API. Use `POST .../telephony-presets/{id}/outbound-call` to
  place calls; subscribe to **Events** (SSE) for call lifecycle.

### 10.5 Multi-tenant SaaS on top of AgenticSAI

- Map *your* tenants → one AgenticSAI **Project** per tenant (hard isolation, separate API
  key, separate usage/limits) **or** one project + per-tenant `user_id`/`dbId` partitioning
  (lighter, shared limits). Prefer project-per-tenant when isolation/billing separation
  matters.

### 10.6 Resilience

- Idempotent retries only (§4.3); circuit-break on sustained `5xx`.
- Recover dropped streams by re-fetching the Run/Trace by id.
- Cache discovery (`event-types`, agent/workflow lists, presets, public config) with TTL.
- Pre-flight `GET /api/public/config` and `GET /health` / `GET /api/v1/.../health`.

---

## 11. Endpoint Reference (by domain)

Every operation in the spec, grouped by domain → tag. Each entry lists auth, parameters, request body schema (expanded one level), and response status/body schemas. Schema names link to §12.

<a id="dom-authentication-identity"></a>

### Authentication & Identity

_15 operations._

#### ▸ Authentication

_Authentication endpoints_

#### `DELETE /api/auth/account`

**Request account deletion**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteAccount`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `GET /api/auth/accountVerification`

**Verify user account**

- **Auth:** **Public** — no authentication required
- **operationId:** `verifyAccount`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | query | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `PUT /api/auth/changePassword`

**Change password with a token**

- **Auth:** **Public** — no authentication required
- **operationId:** `changePassword`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `token` | query | yes | string |  |
| `new_password` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `POST /api/auth/generateResetPasswordLink`

**Generate a password reset link**

- **Auth:** **Public** — no authentication required
- **operationId:** `generateResetPasswordLink`

**Request body** (`application/json`)

Schema: [`ForgotPasswordRequest`](#schema-forgotpasswordrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `email` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `POST /api/auth/google/callback`

**Exchange a Google authorization code for app JWT tokens (GSI redirect flow)**

- **Auth:** **Public** — no authentication required
- **operationId:** `googleCallback`

**Request body** (`application/json`)

Schema: [`GoogleCallbackRequest`](#schema-googlecallbackrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `code` | string | yes |  |  |
| `redirectUri` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LoginResponseDto`](#schema-loginresponsedto) |


#### `POST /api/auth/login`

**Login**

Authenticates user and returns JWT tokens

- **Auth:** **Public** — no authentication required
- **operationId:** `loginDocs`

**Request body** (`application/json`)

Schema: [`LoginRequest`](#schema-loginrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `username` | string | yes |  |  |
| `password` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successful authentication | [`TokenResult`](#schema-tokenresult) |
| `401` | Authentication failed | [`TokenResult`](#schema-tokenresult) |


#### `POST /api/auth/logout`

**Logout the current user**

- **Auth:** `bearer-jwt`
- **operationId:** `logout`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `GET /api/auth/me`

**Get the current user's information**

- **Auth:** `bearer-jwt`
- **operationId:** `me`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`UserDto`](#schema-userdto) |


#### `PUT /api/auth/profile`

**Update the current user's profile**

- **Auth:** `bearer-jwt`
- **operationId:** `updateProfile`

**Request body** (`application/json`)

Schema: [`UpdateProfileRequest`](#schema-updateprofilerequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `firstName` | string |  |  |  |
| `lastName` | string |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`UserDto`](#schema-userdto) |


#### `POST /api/auth/refreshAccessToken`

**Refresh the access token**

- **Auth:** **Public** — no authentication required
- **operationId:** `refreshAccessToken`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `refresh_token` | query | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LoginResponseDto`](#schema-loginresponsedto) |


#### `POST /api/auth/signup`

**Register a new user**

- **Auth:** **Public** — no authentication required
- **operationId:** `register`

**Request body** (`application/json`)

Schema: [`SignUpRequest`](#schema-signuprequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `firstName` | string |  |  |  |
| `lastName` | string |  |  |  |
| `email` | string | yes |  |  |
| `password` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`UserDto`](#schema-userdto) |


#### `PUT /api/auth/updatePassword`

**Update the current user's password**

- **Auth:** `bearer-jwt`
- **operationId:** `updatePassword`

**Request body** (`application/json`)

Schema: [`UpdatePasswordRequest`](#schema-updatepasswordrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `password` | string | yes |  |  |
| `newPassword` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### ▸ Invitations

_Manage invitations endpoints_

#### `POST /api/invitations`

**acceptInvitation_1**

- **Auth:** **Public** — no authentication required
- **operationId:** `acceptInvitation_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `t` | query | yes | string |  |

**Request body** (`application/json`)

Schema: [`UserActivationDto`](#schema-useractivationdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `password` | string | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `GET /api/invitations/{token}`

**getInvitationDetails**

- **Auth:** **Public** — no authentication required
- **operationId:** `getInvitationDetails`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `token` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`InvitationDetails`](#schema-invitationdetails) |


#### `PUT /api/invitations/{token}`

**acceptInvitation**

- **Auth:** **Public** — no authentication required
- **operationId:** `acceptInvitation`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `token` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


---

<a id="dom-organizations-projects-tenancy"></a>

### Organizations, Projects & Tenancy

_50 operations._

#### ▸ Organization Administration

_Administrative endpoints for organization management, monitoring, and rate limit configuration_

#### `GET /api/admin/organizations`

**Get all organizations**

Returns paginated list of all organizations with member and project counts. Supports filtering by organization name (case-insensitive partial match).

- **Auth:** `bearer-jwt`
- **operationId:** `getAllOrganizations`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `name` | query |  | string |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageOrganizationListDto`](#schema-pageorganizationlistdto) |


#### `POST /api/admin/organizations`

**Create a new organization**

Creates a new organization with a default project. The current admin user becomes the organization creator and owner.

- **Auth:** `bearer-jwt`
- **operationId:** `createOrganization_1`

**Request body** (`application/json`)

Schema: [`OrganizationRequest`](#schema-organizationrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Organization created successfully | [`OrganizationDto`](#schema-organizationdto) |
| `400` | Invalid organization data | [`OrganizationDto`](#schema-organizationdto) |
| `403` | Access denied - admin privileges required | [`OrganizationDto`](#schema-organizationdto) |


#### `GET /api/admin/organizations/{organizationId}/rate-limit`

**Get current rate limit configuration**

Returns the active rate limit configuration for an organization

- **Auth:** `bearer-jwt`
- **operationId:** `getRateLimitConfig`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`RateLimitConfigResponseDto`](#schema-ratelimitconfigresponsedto) |


#### `PUT /api/admin/organizations/{organizationId}/rate-limit`

**Update rate limit configuration**

Updates the rate limiting settings for an organization

- **Auth:** `bearer-jwt`
- **operationId:** `updateRateLimitConfig`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`RateLimitConfigRequestDto`](#schema-ratelimitconfigrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `requestsPerMinute` | integer (int64) | yes |  |  |
| `requestsPerHour` | integer (int64) | yes |  |  |
| `requestsPerDay` | integer (int64) | yes |  |  |
| `burstCapacity` | integer (int64) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Rate limit updated successfully | [`RateLimitConfigResponseDto`](#schema-ratelimitconfigresponsedto) |
| `400` | Invalid rate limit configuration | [`RateLimitConfigResponseDto`](#schema-ratelimitconfigresponsedto) |
| `403` | Access denied - admin privileges required | [`RateLimitConfigResponseDto`](#schema-ratelimitconfigresponsedto) |


#### `GET /api/admin/organizations/{organizationId}/usage-summary`

**Get organization usage summary**

Returns aggregated usage overview for an organization including daily trends and top consumers

- **Auth:** `bearer-jwt`
- **operationId:** `getOrganizationUsageSummary`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `startDate` | query | yes | string (date) |  |
| `endDate` | query | yes | string (date) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationUsageSummaryDto`](#schema-organizationusagesummarydto) |


#### ▸ Organizations

_Manage organizations endpoints_

#### `POST /api/organizations`

**createOrganization**

- **Auth:** `bearer-jwt`
- **operationId:** `createOrganization`

**Request body** (`application/json`)

Schema: [`OrganizationRequest`](#schema-organizationrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationDto`](#schema-organizationdto) |


#### `PUT /api/organizations/{organizationId}`

**updateOrganization**

- **Auth:** `bearer-jwt`
- **operationId:** `updateOrganization`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`OrganizationRequest`](#schema-organizationrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationDto`](#schema-organizationdto) |


#### `GET /api/organizations/{organizationId}/limits`

**Get organization limits**

Returns the active rate limits and file upload limits for the organization

- **Auth:** `bearer-jwt`
- **operationId:** `getOrganizationLimits_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationLimitsDto`](#schema-organizationlimitsdto) |


#### `GET /api/organizations/{organizationId}/overview`

**Get organization overview**

Returns summary information about an organization including key counts and usage

- **Auth:** `bearer-jwt`
- **operationId:** `getOrganizationOverview`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `POST /api/organizations/{organizationId}/setup`

**setup**

- **Auth:** `bearer-jwt`
- **operationId:** `setup`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`OrganizationSetupDto`](#schema-organizationsetupdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKeyName` | string | yes |  |  |
| `projectName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiKeySetupResponseDto`](#schema-apikeysetupresponsedto) |


#### `GET /api/organizations/{organizationId}/tags`

**Get organization tags**

Returns all tags for an organization

- **Auth:** `bearer-jwt`
- **operationId:** `getOrganizationTags_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SimpleTagDto`](#schema-simpletagdto)[] |


#### ▸ Organization Projects

_Manage organizations projects endpoints_

#### `GET /api/organizations/{organizationId}/keys`

**listApiKeys_1**

- **Auth:** `bearer-jwt`
- **operationId:** `listApiKeys_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageApiKeyDto`](#schema-pageapikeydto) |


#### `POST /api/organizations/{organizationId}/keys`

**createApiKey_1**

- **Auth:** `bearer-jwt`
- **operationId:** `createApiKey_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`ApiKeyRequest`](#schema-apikeyrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `projectIds` | integer (int64)[] | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiKeyDto`](#schema-apikeydto) |


#### `GET /api/organizations/{organizationId}/keys/system`

**getSystemKey**

- **Auth:** `bearer-jwt`
- **operationId:** `getSystemKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiKeyDto`](#schema-apikeydto) |


#### `POST /api/organizations/{organizationId}/keys/system/regenerate`

**regenerateSystemKey**

- **Auth:** `bearer-jwt`
- **operationId:** `regenerateSystemKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `GET /api/organizations/{organizationId}/keys/{id}`

**getApiKey_1**

- **Auth:** `bearer-jwt`
- **operationId:** `getApiKey_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiKeyDto`](#schema-apikeydto) |


#### `PUT /api/organizations/{organizationId}/keys/{id}`

**updateApiKey_1**

- **Auth:** `bearer-jwt`
- **operationId:** `updateApiKey_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`ApiKeyRequest`](#schema-apikeyrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `projectIds` | integer (int64)[] | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiKeyDto`](#schema-apikeydto) |


#### `DELETE /api/organizations/{organizationId}/keys/{id}`

**revokeApiKey_1**

- **Auth:** `bearer-jwt`
- **operationId:** `revokeApiKey_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `GET /api/organizations/{organizationId}/projects`

**getProjects**

- **Auth:** `bearer-jwt`
- **operationId:** `getProjects`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `archived` | query |  | boolean |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectDto`](#schema-projectdto)[] |


#### `POST /api/organizations/{organizationId}/projects`

**createProject_1**

- **Auth:** `bearer-jwt`
- **operationId:** `createProject_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`BaseProjectDto`](#schema-baseprojectdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectDto`](#schema-projectdto) |


#### `GET /api/organizations/{organizationId}/projects/options`

**getProjectOptions**

- **Auth:** `bearer-jwt`
- **operationId:** `getProjectOptions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectOptionDto`](#schema-projectoptiondto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}`

**getProject_1**

- **Auth:** `bearer-jwt`
- **operationId:** `getProject_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectDto`](#schema-projectdto) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}`

**updateProject_1**

- **Auth:** `bearer-jwt`
- **operationId:** `updateProject_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`BaseProjectDto`](#schema-baseprojectdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectDto`](#schema-projectdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}`

**archiveProject_1**

- **Auth:** `bearer-jwt`
- **operationId:** `archiveProject_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectDto`](#schema-projectdto) |


#### ▸ Project Members

_Manage project members endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/users`

**getProjectMembers**

- **Auth:** `bearer-jwt`
- **operationId:** `getProjectMembers`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `query` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectMemberDto`](#schema-projectmemberdto)[] |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/users`

**inviteProjectMembers**

- **Auth:** `bearer-jwt`
- **operationId:** `inviteProjectMembers`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`InviteProjectMemberDto`](#schema-inviteprojectmemberdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `userIds` | integer (int64)[] |  |  |  |
| `emails` | string[] |  |  |  |
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectMemberDto`](#schema-projectmemberdto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/users/options`

**getUserOptions**

- **Auth:** `bearer-jwt`
- **operationId:** `getUserOptions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`BaseUserOptionDto`](#schema-baseuseroptiondto)[] |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/users/{userId}`

**updateProjectMember**

- **Auth:** `bearer-jwt`
- **operationId:** `updateProjectMember`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`BaseProjectMemberDto`](#schema-baseprojectmemberdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ProjectMemberDto`](#schema-projectmemberdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/users/{userId}`

**deleteProjectMember**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteProjectMember`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/users/{userId}/resend-invitation`

**resendInvitation**

- **Auth:** `bearer-jwt`
- **operationId:** `resendInvitation`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### ▸ Role Presets

_Manage reusable permission templates for organizations and projects._

#### `GET /api/organizations/{organizationId}/role-presets`

**List all role presets**

Returns all role presets for the organization, optionally filtered by type

- **Auth:** `bearer-jwt`
- **operationId:** `list`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `type` | query |  | string | Filter by preset type (ORGANIZATION or PROJECT) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | List of role presets | [`RolePresetResponse`](#schema-rolepresetresponse)[] |
| `403` | Insufficient permissions | [`RolePresetResponse`](#schema-rolepresetresponse)[] |
| `404` | Organization not found | [`RolePresetResponse`](#schema-rolepresetresponse)[] |


#### `POST /api/organizations/{organizationId}/role-presets`

**Create a new role preset**

Creates a new reusable permission template for the organization

- **Auth:** `bearer-jwt`
- **operationId:** `create`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`RolePresetCreateRequest`](#schema-rolepresetcreaterequest)

> Request to create a new role preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | Role preset name |
| `type` | string | yes | `ORGANIZATION`, `PROJECT` | Role preset type (ORGANIZATION or PROJECT) |
| `permissions` | object | yes |  | Permissions configuration as map - converted to OrganizationPermissions or ProjectPermissions based on type |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Role preset created successfully | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `400` | Invalid request data | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `403` | Insufficient permissions | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `404` | Organization not found | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `409` | Role preset with this name already exists | [`RolePresetResponse`](#schema-rolepresetresponse) |


#### `GET /api/organizations/{organizationId}/role-presets/{presetId}`

**Get a role preset by ID**

Returns a specific role preset

- **Auth:** `bearer-jwt`
- **operationId:** `getById`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `presetId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Role preset details | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `403` | Insufficient permissions | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `404` | Role preset or organization not found | [`RolePresetResponse`](#schema-rolepresetresponse) |


#### `PUT /api/organizations/{organizationId}/role-presets/{presetId}`

**Update a role preset**

Updates an existing role preset. Note: type cannot be changed after creation.

- **Auth:** `bearer-jwt`
- **operationId:** `update`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `presetId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`RolePresetUpdateRequest`](#schema-rolepresetupdaterequest)

> Request to update an existing role preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | Role preset name |
| `permissions` | object | yes |  | Permissions configuration as map - converted based on preset type |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Role preset updated successfully | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `400` | Invalid request data | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `403` | Insufficient permissions | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `404` | Role preset or organization not found | [`RolePresetResponse`](#schema-rolepresetresponse) |
| `409` | Role preset with this name already exists | [`RolePresetResponse`](#schema-rolepresetresponse) |


#### `DELETE /api/organizations/{organizationId}/role-presets/{presetId}`

**Delete a role preset**

Deletes a role preset from the organization

- **Auth:** `bearer-jwt`
- **operationId:** `delete`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `presetId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Role preset deleted successfully |  |
| `403` | Insufficient permissions |  |
| `404` | Role preset or organization not found |  |


#### ▸ Organization Secrets

_Org-scoped secrets management endpoints_

#### `GET /api/organizations/{organizationId}/secrets`

**listSecrets**

- **Auth:** `bearer-jwt`
- **operationId:** `listSecrets`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `key` | query |  | string |  |
| `description` | query |  | string |  |
| `page` | query |  | integer (int32) |  |
| `size` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageSecretResponse`](#schema-pagesecretresponse) |


#### `POST /api/organizations/{organizationId}/secrets`

**createSecret**

- **Auth:** `bearer-jwt`
- **operationId:** `createSecret`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`CreateSecretRequest`](#schema-createsecretrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string | yes | minLength=`0` ; maxLength=`255` |  |
| `value` | string | yes | minLength=`1` |  |
| `description` | string |  | minLength=`0` ; maxLength=`1000` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SecretResponse`](#schema-secretresponse) |


#### `GET /api/organizations/{organizationId}/secrets/{id}`

**getSecret**

- **Auth:** `bearer-jwt`
- **operationId:** `getSecret`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SecretResponse`](#schema-secretresponse) |


#### `PATCH /api/organizations/{organizationId}/secrets/{id}`

**updateSecret**

- **Auth:** `bearer-jwt`
- **operationId:** `updateSecret`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`UpdateSecretRequest`](#schema-updatesecretrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string |  | minLength=`0` ; maxLength=`255` |  |
| `value` | string |  |  |  |
| `description` | string |  | minLength=`0` ; maxLength=`1000` |  |
| `isEmpty` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SecretResponse`](#schema-secretresponse) |


#### `DELETE /api/organizations/{organizationId}/secrets/{id}`

**deleteSecret**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteSecret`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### ▸ Organization Members

_Manage organizations members endpoints_

#### `GET /api/organizations/{organizationId}/users`

**getMembers-CL8VjLI**

- **Auth:** `bearer-jwt`
- **operationId:** `getMembers-CL8VjLI`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `query` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationMemberDto`](#schema-organizationmemberdto)[] |


#### `PUT /api/organizations/{organizationId}/users`

**inviteMembers-CL8VjLI**

- **Auth:** `bearer-jwt`
- **operationId:** `inviteMembers-CL8VjLI`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`InviteOrgMembersDto`](#schema-inviteorgmembersdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `emails` | string[] | yes |  |  |
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`BaseUserDto`](#schema-baseuserdto)[] |


#### `PUT /api/organizations/{organizationId}/users/{userId}`

**updateMember-ShzOSBw**

- **Auth:** `bearer-jwt`
- **operationId:** `updateMember-ShzOSBw`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`BaseOrganizationMemberDto`](#schema-baseorganizationmemberdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`OrganizationMemberDto`](#schema-organizationmemberdto) |


#### `DELETE /api/organizations/{organizationId}/users/{userId}`

**deleteMember-EVQb1pU**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteMember-EVQb1pU`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `POST /api/organizations/{organizationId}/users/{userId}/resend-invitation`

**resendInvitation-EVQb1pU**

- **Auth:** `bearer-jwt`
- **operationId:** `resendInvitation-EVQb1pU`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `userId` | path | yes | integer (int64) |  |
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### ▸ Projects v1

_Project management endpoints with system API key authentication_

#### `GET /client/api/v1/projects`

**List projects**

Returns all projects for the organization

- **Auth:** `api-key`
- **operationId:** `listProjects`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `archived` | query |  | boolean |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseListProjectDto`](#schema-apiresponselistprojectdto) |


#### `POST /client/api/v1/projects`

**Create project**

Creates a new project in the organization

- **Auth:** `api-key`
- **operationId:** `createProject`

**Request body** (`application/json`)

Schema: [`BaseProjectDto`](#schema-baseprojectdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseProjectDto`](#schema-apiresponseprojectdto) |


#### `GET /client/api/v1/projects/{projectRefId}`

**Get project**

Returns a single project by reference ID

- **Auth:** `api-key`
- **operationId:** `getProject`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseProjectDto`](#schema-apiresponseprojectdto) |


#### `PUT /client/api/v1/projects/{projectRefId}`

**Update project**

Updates a project's name

- **Auth:** `api-key`
- **operationId:** `updateProject`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`BaseProjectDto`](#schema-baseprojectdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseProjectDto`](#schema-apiresponseprojectdto) |


#### `DELETE /client/api/v1/projects/{projectRefId}`

**Archive project**

Archives (soft-deletes) a project

- **Auth:** `api-key`
- **operationId:** `archiveProject`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseProjectDto`](#schema-apiresponseprojectdto) |


---

<a id="dom-agents"></a>

### Agents

_39 operations._

#### ▸ Agent Creator

_Create and update agents with AI assistance_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/agent-creator`

**getCreatorAgent**

- **Auth:** `bearer-jwt`
- **operationId:** `getCreatorAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentResponse`](#schema-agentresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agent-creator/agent`

**createAgentDirect**

- **Auth:** `bearer-jwt`
- **operationId:** `createAgentDirect`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`CreateAgentDirectRequest`](#schema-createagentdirectrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `initialMessage` | string | yes | minLength=`10` ; maxLength=`1000` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentResponse`](#schema-agentresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agent-creator/create`

**createWithAssistant**

- **Auth:** `bearer-jwt`
- **operationId:** `createWithAssistant`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`CreateWithAssistantRequest`](#schema-createwithassistantrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string |  | minLength=`10` ; maxLength=`1000` |  |
| `provider` | integer (int64) | yes |  |  |
| `modelName` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `creatorOnly` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentResponse`](#schema-agentresponse) |


#### ▸ Agent Management

_Agent CRUD operations_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents`

**List agents for a project**

Retrieves all agents belonging to a specific project with pagination and filtering support. Use query parameters: page (0-indexed), size (default 20), sort (e.g., 'name,asc' or 'createdAt,desc'), name (optional filter by name, case-insensitive partial match)

- **Auth:** `bearer-jwt`
- **operationId:** `listAgents_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `name` | query |  | string | Filter by agent name (case-insensitive, partial match) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agents retrieved successfully | [`PageAgentResponse`](#schema-pageagentresponse) |
| `403` | User does not have permission to view agents in this project | [`PageAgentResponse`](#schema-pageagentresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents`

**Create a new agent**

Creates a new AI agent with specified configuration. The agent is created both in the database and in the AI service.

- **Auth:** `bearer-jwt`
- **operationId:** `createAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Request body** (`application/json`)

Schema: [`AgentCreateRequest`](#schema-agentcreaterequest)

> Request to create a new agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Agent created successfully | [`AgentResponse`](#schema-agentresponse) |
| `400` | Invalid request data | [`AgentResponse`](#schema-agentresponse) |
| `403` | User does not have permission to create agents in this project | [`AgentResponse`](#schema-agentresponse) |
| `404` | Organization or project not found | [`AgentResponse`](#schema-agentresponse) |
| `503` | AI service unavailable | [`AgentResponse`](#schema-agentresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/llm-providers`

**Get configured LLM providers for agents**

Retrieves all LLM provider configurations for the organization with their available models. Models are filtered based on the enabled and custom models configured for each provider.

- **Auth:** `bearer-jwt`
- **operationId:** `getConfiguredLlmProviders_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Providers retrieved successfully | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |
| `403` | User does not have permission to view providers | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |
| `404` | Organization not found | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/options`

**agentOptions_1**

- **Auth:** `bearer-jwt`
- **operationId:** `agentOptions_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`BaseAgentOptionDto`](#schema-baseagentoptiondto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/providers`

**Get supported AI service providers**

- **Auth:** `bearer-jwt`
- **operationId:** `getAiServiceProviders`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Agent deleted successfully | [`AIProviderDto`](#schema-aiproviderdto)[] |
| `403` | User does not have permission to delete this agent | [`AIProviderDto`](#schema-aiproviderdto)[] |
| `404` | Agent not found | [`AIProviderDto`](#schema-aiproviderdto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}`

**Get agent by ID**

Retrieves agent details by AI service agent ID

- **Auth:** `bearer-jwt`
- **operationId:** `getAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent found | [`AgentResponse`](#schema-agentresponse) |
| `403` | User does not have permission to view this agent | [`AgentResponse`](#schema-agentresponse) |
| `404` | Agent not found | [`AgentResponse`](#schema-agentresponse) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}`

**Update an agent**

Updates an existing agent's name, description, and/or configuration

- **Auth:** `bearer-jwt`
- **operationId:** `updateAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID |

**Request body** (`application/json`)

Schema: [`AgentUpdateRequest`](#schema-agentupdaterequest)

> Request to update an existing agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent updated successfully | [`AgentResponse`](#schema-agentresponse) |
| `400` | Invalid request data | [`AgentResponse`](#schema-agentresponse) |
| `403` | User does not have permission to update this agent | [`AgentResponse`](#schema-agentresponse) |
| `404` | Agent not found | [`AgentResponse`](#schema-agentresponse) |
| `503` | AI service unavailable | [`AgentResponse`](#schema-agentresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}`

**Delete an agent**

Soft deletes an agent. The agent is marked as deleted in the database and deletion is attempted in the AI service (best effort).

- **Auth:** `bearer-jwt`
- **operationId:** `deleteAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Agent deleted successfully |  |
| `403` | User does not have permission to delete this agent |  |
| `404` | Agent not found |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/clone`

**Clone an agent**

Creates a copy of an existing agent with a new name, inheriting all configuration from the source agent.

- **Auth:** `bearer-jwt`
- **operationId:** `cloneAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID to clone |

**Request body** (`application/json`)

Schema: [`AgentCloneRequest`](#schema-agentclonerequest)

> Request to clone an existing agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned agent |
| `presetName` | string |  | minLength=`1` ; maxLength=`100` | Display name for the cloned agent shown in the platform dashboard |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Agent cloned successfully | [`AgentResponse`](#schema-agentresponse) |
| `400` | Invalid request data | [`AgentResponse`](#schema-agentresponse) |
| `403` | User does not have permission to create agents in this project | [`AgentResponse`](#schema-agentresponse) |
| `404` | Source agent not found | [`AgentResponse`](#schema-agentresponse) |
| `503` | AI service unavailable | [`AgentResponse`](#schema-agentresponse) |


#### ▸ Agent Memories

_Retrieve and manage agent memory endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/memories`

**listMemories_1**

- **Auth:** `bearer-jwt`
- **operationId:** `listMemories_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `searchContent` | query |  | string |  |
| `topics` | query |  | string[] |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageMemoryListDto`](#schema-pagememorylistdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/memories/{memoryId}`

**deleteMemory_1**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteMemory_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `memoryId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### ▸ Agent Execution

_Agent execution endpoints (streaming and non-streaming)_

#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run`

**Execute agent (non-streaming)**

Executes an agent and returns the complete response with metrics. Usage is automatically tracked.

- **Auth:** `bearer-jwt`
- **operationId:** `executeAgent_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "message": {
      "type": "string",
      "description": "User message/prompt for the agent"
    },
    "sessionId": {
      "type": "string",
      "description": "Optional session ID for conversation continuity"
    },
    "toolContext": {
      "type": "string",
      "description": "Optional contextual data for MCP tool parameter mapping (JSON string)"
    },
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  },
  "required": [
    "message"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent executed successfully | [`AgentExecutionResponse`](#schema-agentexecutionresponse) |
| `400` | Invalid request data | [`AgentExecutionResponse`](#schema-agentexecutionresponse) |
| `403` | User does not have permission to execute this agent | [`AgentExecutionResponse`](#schema-agentexecutionresponse) |
| `404` | Agent not found | [`AgentExecutionResponse`](#schema-agentexecutionresponse) |
| `503` | AI service unavailable | [`AgentExecutionResponse`](#schema-agentexecutionresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run/stream`

**Execute agent (streaming)**

Executes an agent and streams the response via Server-Sent Events (SSE). Usage is tracked when metrics are received.

- **Auth:** `bearer-jwt`
- **operationId:** `executeAgentStreaming`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `agentId` | path | yes | string | AI service agent ID |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "message": {
      "type": "string",
      "description": "User message/prompt for the agent"
    },
    "sessionId": {
      "type": "string",
      "description": "Optional session ID for conversation continuity"
    },
    "toolContext": {
      "type": "string",
      "description": "Optional contextual data for MCP tool parameter mapping (JSON string)"
    },
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  },
  "required": [
    "message"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent execution stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid request data | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `403` | User does not have permission to execute this agent | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Agent not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI service unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/runs/{runId}/cancel`

**Cancel an agent run**

Cancels an in-progress agent run. Usage is tracked after successful cancellation.

- **Auth:** `bearer-jwt`
- **operationId:** `cancelAgentRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `runId` | path | yes | string | AI service run ID |
| `agentId` | path | yes | string | AI service agent ID |
| `sessionId` | query | yes | string | AI service session ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully | `string` |
| `403` | User does not have permission to execute this agent | `string` |
| `404` | Agent or run not found | `string` |
| `503` | AI service unavailable | `string` |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/runs/{runId}/continue`

**Continue a paused agent run (streaming)**

Resumes a paused agent run — typically after a tool-approval step — and streams the response via Server-Sent Events (SSE).

- **Auth:** `bearer-jwt`
- **operationId:** `continueAgentRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `runId` | path | yes | string | AI service run ID |
| `agentId` | path | yes | string | AI service agent ID |

**Request body** (`application/json`)

Schema: [`ContinueAgentRunRequest`](#schema-continueagentrunrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionId` | string | yes |  | Session ID for conversation continuity |
| `tools` | string |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Continuation stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid request data | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `403` | User does not have permission to execute this agent | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Agent or run not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI service unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### ▸ Agent Sessions

_Manage agent session endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions`

**listSessions_5**

- **Auth:** `bearer-jwt`
- **operationId:** `listSessions_5`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `sessionName` | query |  | string |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageSessionListDto`](#schema-pagesessionlistdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}`

**getSession_5**

- **Auth:** `bearer-jwt`
- **operationId:** `getSession_5`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}`

**deleteSession_5**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteSession_5`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}/rename`

**renameSession_5**

- **Auth:** `bearer-jwt`
- **operationId:** `renameSession_5`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}/runs`

**getSessionRuns_5**

- **Auth:** `bearer-jwt`
- **operationId:** `getSessionRuns_5`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionRunDto`](#schema-sessionrundto)[] |


#### ▸ Agent Versions

_Agent version history management_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions`

**listVersions**

- **Auth:** `bearer-jwt`
- **operationId:** `listVersions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `aiServiceAgentId` | path | yes | string |  |
| `page` | query |  | integer (int32) |  |
| `size` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageAgentVersionResponse`](#schema-pageagentversionresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/compare`

**compareVersions**

- **Auth:** `bearer-jwt`
- **operationId:** `compareVersions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `aiServiceAgentId` | path | yes | string |  |
| `fromVersionId` | query | yes | integer (int64) |  |
| `toVersionId` | query | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentVersionComparisonResponse`](#schema-agentversioncomparisonresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/{versionId}`

**getVersionDetail**

- **Auth:** `bearer-jwt`
- **operationId:** `getVersionDetail`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `aiServiceAgentId` | path | yes | string |  |
| `versionId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentVersionResponse`](#schema-agentversionresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/{versionId}/rollback`

**rollbackToVersion**

- **Auth:** `bearer-jwt`
- **operationId:** `rollbackToVersion`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `aiServiceAgentId` | path | yes | string |  |
| `versionId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentVersionResponse`](#schema-agentversionresponse) |


#### ▸ Runs

_Agent runs per project_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/logs`

**Get agent runs**

Returns paginated list of agent runs with filtering. Sortable fields: created/created_ts, id, inputTokens/input_tokens, outputTokens/output_tokens, duration/duration_seconds

- **Auth:** `bearer-jwt`
- **operationId:** `getAgentRuns`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `status` | query |  | string |  |
| `scope` | query |  | string |  |
| `usageMode` | query |  | string |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |
| `userIds` | query |  | integer (int64)[] |  |
| `apiKeyIds` | query |  | integer (int64)[] |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageAgentRunDto`](#schema-pageagentrundto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/logs/{runId}`

**Get agent runs details**

Returns detailed execution data for a specific run

- **Auth:** `bearer-jwt`
- **operationId:** `getAgentRunDetail`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `runId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### ▸ Agent Creator Service

_Endpoints for AI Service (Agent Creator) access. Required header: 'X-Service-Key' (agent-creator category). Optional: 'X-Service-Name'_

#### `PUT /api/service/agents/{aiServiceAgentId}`

**updateAgentConfiguration**

- **Auth:** `service-key-agent-creator` or `service-name`
- **operationId:** `updateAgentConfiguration`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `aiServiceAgentId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`AgentConfigurationUpdateRequest`](#schema-agentconfigurationupdaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`1` ; maxLength=`100` |  |
| `presetName` | string |  | minLength=`1` ; maxLength=`100` |  |
| `description` | string |  | minLength=`10` ; maxLength=`1000` |  |
| `configuration` | [`AgentConfigurationString`](#schema-agentconfigurationstring) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `GET /api/service/agents/{aiServiceAgentId}/http-requests`

**getAgentHttpRequests**

- **Auth:** `service-key-agent-creator` or `service-name`
- **operationId:** `getAgentHttpRequests`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `aiServiceAgentId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`HttpRequestResponse`](#schema-httprequestresponse)[] |


#### `GET /api/service/agents/{aiServiceAgentId}/llm-providers`

**getAgentLlmProviders**

- **Auth:** `service-key-agent-creator` or `service-name`
- **operationId:** `getAgentLlmProviders`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `aiServiceAgentId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |


#### `GET /api/service/agents/{aiServiceAgentId}/mcp-servers`

**getAgentMcpServers**

- **Auth:** `service-key-agent-creator` or `service-name`
- **operationId:** `getAgentMcpServers`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `aiServiceAgentId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`MCPServerWithToolsResponse`](#schema-mcpserverwithtoolsresponse)[] |


#### `GET /api/service/agents/{aiServiceAgentId}/storage-resources`

**getAgentStorageResources**

- **Auth:** `service-key-agent-creator` or `service-name`
- **operationId:** `getAgentStorageResources`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `aiServiceAgentId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`StorageResourceResponse`](#schema-storageresourceresponse)[] |


#### ▸ Agent Sessions v1

_Agent session management with API key authentication_

#### `GET /client/api/v1/agents/{agentId}/sessions`

**List agent sessions**

Returns a paginated list of sessions for the specified agent.
            Sessions are filtered by the authenticated API key's organization and project.
            Optionally filter by session name.

- **Auth:** `api-key`
- **operationId:** `listSessions_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |
| `user_id` | query | yes | string | Filter sessions by user ID |
| `sessionName` | query |  | string | Filter sessions by name (optional) |
| `limit` | query |  | integer (int32) | Number of results per page |
| `page` | query |  | integer (int32) | Page number (1-based) |
| `sortBy` | query |  | string | Sort field |
| `sortOrder` | query |  | string | Sort order (asc/desc) |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Sessions retrieved successfully | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `400` | Invalid request parameters | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `401` | Invalid or missing API key | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `404` | Agent not found or doesn't belong to project | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `429` | Rate limit exceeded | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |


#### `GET /client/api/v1/agents/{agentId}/sessions/{sessionId}`

**Get agent session by ID**

Retrieves detailed information about a specific session

- **Auth:** `api-key`
- **operationId:** `getSession_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session retrieved successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Agent or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `DELETE /client/api/v1/agents/{agentId}/sessions/{sessionId}`

**Delete agent session**

Deletes a specific session for the agent

- **Auth:** `api-key`
- **operationId:** `deleteSession_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Session deleted successfully |  |
| `401` | Invalid or missing API key |  |
| `404` | Agent or session not found |  |


#### `POST /client/api/v1/agents/{agentId}/sessions/{sessionId}/rename`

**Rename agent session**

Updates the name of a specific session

- **Auth:** `api-key`
- **operationId:** `renameSession_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session renamed successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `400` | Invalid request data | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Agent or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `GET /client/api/v1/agents/{agentId}/sessions/{sessionId}/runs`

**Get agent session runs**

Returns all runs (execution history) for a specific session

- **Auth:** `api-key`
- **operationId:** `getSessionRuns_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session runs retrieved successfully | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `401` | Invalid or missing API key | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `404` | Agent or session not found | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |


---

<a id="dom-teams"></a>

### Teams

_17 operations._

#### ▸ Teams

_Manage AI teams_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams`

**List teams for a project**

Retrieves all teams belonging to a specific project with pagination and optional name filtering

- **Auth:** `bearer-jwt`
- **operationId:** `listTeams_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `name` | query |  | string | Filter by team name (case-insensitive, partial match) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Teams retrieved successfully | [`PageTeamResponse`](#schema-pageteamresponse) |
| `403` | User does not have permission to view teams in this project | [`PageTeamResponse`](#schema-pageteamresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams`

**Create a new team**

Creates a new AI team with multi-agent configuration. The team is created both in the database and in the AI service.

- **Auth:** `bearer-jwt`
- **operationId:** `createTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Request body** (`application/json`)

Schema: [`TeamCreateRequest`](#schema-teamcreaterequest)

> Request to create a new team

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name known to the AI (e.g. 'Support Team') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Support') |
| `projectId` | integer (int64) |  |  | ID of the project this team belongs to. Required when calling V1 API (/client/api/v1/teams). |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) | yes |  | Team configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Team created successfully | [`TeamResponse`](#schema-teamresponse) |
| `400` | Invalid request data or validation failed | [`TeamResponse`](#schema-teamresponse) |
| `403` | User does not have permission to create teams in this project | [`TeamResponse`](#schema-teamresponse) |
| `404` | Organization, project, or referenced agent/storage not found | [`TeamResponse`](#schema-teamresponse) |
| `503` | AI service unavailable | [`TeamResponse`](#schema-teamresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/options`

**listTeamOptions_1**

- **Auth:** `bearer-jwt`
- **operationId:** `listTeamOptions_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`BaseAgentOptionDto`](#schema-baseagentoptiondto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}`

**Get team by ID**

Retrieves team details by AI service team ID

- **Auth:** `bearer-jwt`
- **operationId:** `getTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team found | [`TeamResponse`](#schema-teamresponse) |
| `403` | User does not have permission to view this team | [`TeamResponse`](#schema-teamresponse) |
| `404` | Team not found | [`TeamResponse`](#schema-teamresponse) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}`

**Update a team**

Updates an existing team's name, description, and/or configuration

- **Auth:** `bearer-jwt`
- **operationId:** `updateTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`application/json`)

Schema: [`TeamUpdateRequest`](#schema-teamupdaterequest)

> Request to update an existing team (all fields optional)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name (optional - only updates if provided) |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description (optional - only updates if provided) |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) |  |  | Team configuration (optional - only updates if provided) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team updated successfully | [`TeamResponse`](#schema-teamresponse) |
| `400` | Invalid request data | [`TeamResponse`](#schema-teamresponse) |
| `403` | User does not have permission to update this team | [`TeamResponse`](#schema-teamresponse) |
| `404` | Team not found | [`TeamResponse`](#schema-teamresponse) |
| `503` | AI service unavailable | [`TeamResponse`](#schema-teamresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}`

**Delete a team**

Soft deletes a team. The team is marked as deleted in the database and deletion is attempted in the AI service (best effort).

- **Auth:** `bearer-jwt`
- **operationId:** `deleteTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Team deleted successfully |  |
| `403` | User does not have permission to delete this team |  |
| `404` | Team not found |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/clone`

**Clone a team**

Creates a copy of an existing team with a new name

- **Auth:** `bearer-jwt`
- **operationId:** `cloneTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`application/json`)

Schema: [`TeamCloneRequest`](#schema-teamclonerequest)

> Request to clone a team

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned team |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name for the cloned team shown in the platform dashboard |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Team cloned successfully | [`TeamResponse`](#schema-teamresponse) |
| `400` | Invalid request data | [`TeamResponse`](#schema-teamresponse) |
| `403` | User does not have permission to clone this team | [`TeamResponse`](#schema-teamresponse) |
| `404` | Team not found | [`TeamResponse`](#schema-teamresponse) |
| `503` | AI service unavailable | [`TeamResponse`](#schema-teamresponse) |


#### ▸ Team Memories

_Retrieve and manage team memory endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/memories`

**listMemories**

- **Auth:** `bearer-jwt`
- **operationId:** `listMemories`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `searchContent` | query |  | string |  |
| `topics` | query |  | string[] |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageMemoryListDto`](#schema-pagememorylistdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/memories/{memoryId}`

**deleteMemory**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteMemory`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `memoryId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### ▸ Team Execution

_Team execution endpoints (streaming and non-streaming)_

#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run`

**Execute team (non-streaming)**

Executes a team and returns the complete response with metrics. Usage is automatically tracked.

- **Auth:** `bearer-jwt`
- **operationId:** `executeTeam_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "message": {
      "type": "string",
      "description": "User message/prompt for the team"
    },
    "sessionId": {
      "type": "string",
      "description": "Optional session ID for conversation continuity"
    },
    "monitor": {
      "type": "boolean",
      "description": "Enable monitoring (default: true)"
    },
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  },
  "required": [
    "message"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team executed successfully | [`TeamExecutionResponse`](#schema-teamexecutionresponse) |
| `400` | Invalid request data | [`TeamExecutionResponse`](#schema-teamexecutionresponse) |
| `403` | User does not have permission to execute this team | [`TeamExecutionResponse`](#schema-teamexecutionresponse) |
| `404` | Team not found | [`TeamExecutionResponse`](#schema-teamexecutionresponse) |
| `503` | AI service unavailable | [`TeamExecutionResponse`](#schema-teamexecutionresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run/stream`

**Execute team (streaming)**

Executes a team and streams the response via Server-Sent Events (SSE). Usage is tracked when metrics are received.

- **Auth:** `bearer-jwt`
- **operationId:** `executeTeamStreaming`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "message": {
      "type": "string",
      "description": "User message/prompt for the team"
    },
    "sessionId": {
      "type": "string",
      "description": "Optional session ID for conversation continuity"
    },
    "monitor": {
      "type": "boolean",
      "description": "Enable monitoring (default: true)"
    },
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  },
  "required": [
    "message"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team execution stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid request data | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `403` | User does not have permission to execute this team | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Team not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI service unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/runs/{runId}/cancel`

**Cancel a team run**

Cancels an in-progress team run. Usage is tracked after successful cancellation.

- **Auth:** `bearer-jwt`
- **operationId:** `cancelTeamRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `runId` | path | yes | string | AI service run ID |
| `teamId` | path | yes | string | AI service team ID |
| `sessionId` | query | yes | string | AI service session ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully | `string` |
| `403` | User does not have permission to execute this team | `string` |
| `404` | Team or run not found | `string` |
| `503` | AI service unavailable | `string` |


#### ▸ Team Sessions

_Manage team session endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions`

**listSessions_4**

- **Auth:** `bearer-jwt`
- **operationId:** `listSessions_4`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `sessionName` | query |  | string |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageSessionListDto`](#schema-pagesessionlistdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}`

**getSession_4**

- **Auth:** `bearer-jwt`
- **operationId:** `getSession_4`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}`

**deleteSession_4**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteSession_4`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}/rename`

**renameSession_4**

- **Auth:** `bearer-jwt`
- **operationId:** `renameSession_4`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}/runs`

**getSessionRuns_4**

- **Auth:** `bearer-jwt`
- **operationId:** `getSessionRuns_4`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionRunDto`](#schema-sessionrundto)[] |


---

<a id="dom-workflows"></a>

### Workflows

_15 operations._

#### ▸ Workflow Management

_Workflow discovery and details operations_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/workflows`

**List available workflows**

Retrieves all workflows available from the AI service. Workflows are predefined processes that can handle specific tasks like OCR, document processing, etc.

- **Auth:** `bearer-jwt`
- **operationId:** `listWorkflows_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflows retrieved successfully | [`AiWorkflowListItem`](#schema-aiworkflowlistitem)[] |
| `403` | User does not have permission to view workflows | [`AiWorkflowListItem`](#schema-aiworkflowlistitem)[] |
| `503` | AI service unavailable | [`AiWorkflowListItem`](#schema-aiworkflowlistitem)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}`

**Get workflow details**

Retrieves detailed information about a specific workflow, including its input schema and configuration.

- **Auth:** `bearer-jwt`
- **operationId:** `getWorkflowDetails_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `workflowId` | path | yes | string | Workflow ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflow details retrieved successfully | [`AiWorkflowDetailsResponse`](#schema-aiworkflowdetailsresponse) |
| `403` | User does not have permission to view workflow details | [`AiWorkflowDetailsResponse`](#schema-aiworkflowdetailsresponse) |
| `404` | Workflow not found | [`AiWorkflowDetailsResponse`](#schema-aiworkflowdetailsresponse) |
| `503` | AI service unavailable | [`AiWorkflowDetailsResponse`](#schema-aiworkflowdetailsresponse) |


#### ▸ Workflow Execution

_Workflow execution endpoints (streaming and non-streaming)_

#### `POST /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run`

**Execute workflow (non-streaming)**

Executes a workflow and returns the complete response. Accepts dynamic form data where non-file fields are combined into a JSON message. Usage is automatically tracked.

- **Auth:** `bearer-jwt`
- **operationId:** `executeWorkflow_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `workflowId` | path | yes | string | Workflow ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflow executed successfully | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `400` | Invalid request data | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `403` | User does not have permission to execute this workflow | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `404` | Workflow not found | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `503` | AI service unavailable | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run/stream`

**Execute workflow (streaming)**

Executes a workflow and streams the response via Server-Sent Events (SSE). Accepts dynamic form data where non-file fields are combined into a JSON message.

- **Auth:** `bearer-jwt`
- **operationId:** `executeWorkflowStreaming`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `workflowId` | path | yes | string | Workflow ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflow execution stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid request data | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `403` | User does not have permission to execute this workflow | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Workflow not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI service unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/runs/{runId}/cancel`

**Cancel a workflow run**

Cancels an in-progress workflow run. Usage is tracked after successful cancellation.

- **Auth:** `bearer-jwt`
- **operationId:** `cancelWorkflowRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `workflowId` | path | yes | string | Workflow ID |
| `runId` | path | yes | string | Run ID |
| `sessionId` | query | yes | string | Session ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully |  |
| `403` | User does not have permission to execute this workflow |  |
| `404` | Workflow or run not found |  |
| `503` | AI service unavailable |  |


#### ▸ Workflow Sessions

_Manage workflow session endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions`

**listSessions_3**

- **Auth:** `bearer-jwt`
- **operationId:** `listSessions_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `sessionName` | query |  | string |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageSessionListDto`](#schema-pagesessionlistdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}`

**getSession_3**

- **Auth:** `bearer-jwt`
- **operationId:** `getSession_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}`

**deleteSession_3**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteSession_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}/rename`

**renameSession_3**

- **Auth:** `bearer-jwt`
- **operationId:** `renameSession_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionDto`](#schema-sessiondto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}/runs`

**getSessionRuns_3**

- **Auth:** `bearer-jwt`
- **operationId:** `getSessionRuns_3`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionRunDto`](#schema-sessionrundto)[] |


#### ▸ Workflow Sessions v1

_Workflow session management with API key authentication_

#### `GET /client/api/v1/workflows/{workflowId}/sessions`

**List workflow sessions**

Returns a paginated list of sessions for the specified workflow.
            Sessions are filtered by the authenticated API key's organization and project.
            Optionally filter by session name.

- **Auth:** `api-key`
- **operationId:** `listSessions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string | Workflow ID |
| `user_id` | query | yes | string | Filter sessions by user ID |
| `sessionName` | query |  | string | Filter sessions by name (optional) |
| `limit` | query |  | integer (int32) | Number of results per page |
| `page` | query |  | integer (int32) | Page number (1-based) |
| `sortBy` | query |  | string | Sort field |
| `sortOrder` | query |  | string | Sort order (asc/desc) |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Sessions retrieved successfully | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `400` | Invalid request parameters | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `401` | Invalid or missing API key | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `404` | Workflow not found or doesn't belong to project | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `429` | Rate limit exceeded | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |


#### `GET /client/api/v1/workflows/{workflowId}/sessions/{sessionId}`

**Get workflow session by ID**

Retrieves detailed information about a specific session

- **Auth:** `api-key`
- **operationId:** `getSession`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string | Workflow ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session retrieved successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Workflow or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `DELETE /client/api/v1/workflows/{workflowId}/sessions/{sessionId}`

**Delete workflow session**

Deletes a specific session for the workflow

- **Auth:** `api-key`
- **operationId:** `deleteSession`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string | Workflow ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Session deleted successfully |  |
| `401` | Invalid or missing API key |  |
| `404` | Workflow or session not found |  |


#### `POST /client/api/v1/workflows/{workflowId}/sessions/{sessionId}/rename`

**Rename workflow session**

Updates the name of a specific session

- **Auth:** `api-key`
- **operationId:** `renameSession`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string | Workflow ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session renamed successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `400` | Invalid request data | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Workflow or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `GET /client/api/v1/workflows/{workflowId}/sessions/{sessionId}/runs`

**Get workflow session runs**

Returns all runs (execution history) for a specific session

- **Auth:** `api-key`
- **operationId:** `getSessionRuns`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string | Workflow ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session runs retrieved successfully | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `401` | Invalid or missing API key | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `404` | Workflow or session not found | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |


---

<a id="dom-audio-voice-telephony"></a>

### Audio, Voice & Telephony

_24 operations._

#### ▸ Events

_Subscribe to service sent events_

#### `GET /api/events`

**streamEvents**

- **Auth:** `bearer-jwt`
- **operationId:** `streamEvents`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventMapStringObject`](#schema-serversenteventmapstringobject)[] |


#### `GET /api/events/test`

**testStreamEvents**

- **Auth:** `bearer-jwt`
- **operationId:** `testStreamEvents`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### ▸ Telephony Presets

_Telephony preset management endpoints_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/telephony-presets`

**List telephony presets**

List all telephony presets for a project with pagination and optional filtering by name and type

- **Auth:** **Public** — no authentication required
- **operationId:** `listTelephonyPresets_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `name` | query |  | string | Filter by name (case-insensitive, partial match) |
| `type` | query |  | string | Filter by type (INBOUND or OUTBOUND) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageTelephonyPresetResponseDto`](#schema-pagetelephonypresetresponsedto) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/telephony-presets`

**Create telephony preset**

Create a new telephony preset with SIP trunk configuration

- **Auth:** **Public** — no authentication required
- **operationId:** `createTelephonyPreset_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Request body** (`application/json`)

Schema: [`CreateTelephonyPresetDto`](#schema-createtelephonypresetdto)

> Request DTO for creating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Created | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}`

**Get telephony preset**

Retrieve a telephony preset by ID

- **Auth:** **Public** — no authentication required
- **operationId:** `getTelephonyPreset_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `id` | path | yes | integer (int64) | Telephony preset ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}`

**Update telephony preset**

Update an existing telephony preset

- **Auth:** **Public** — no authentication required
- **operationId:** `updateTelephonyPreset_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `id` | path | yes | integer (int64) | Telephony preset ID |

**Request body** (`application/json`)

Schema: [`UpdateTelephonyPresetDto`](#schema-updatetelephonypresetdto)

> Request DTO for updating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}`

**Delete telephony preset**

Delete a telephony preset

- **Auth:** **Public** — no authentication required
- **operationId:** `deleteTelephonyPreset_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `id` | path | yes | integer (int64) | Telephony preset ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}/outbound-call`

**Create outbound call**

Initiate an outbound phone call using this telephony preset

- **Auth:** **Public** — no authentication required
- **operationId:** `createOutboundCall_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `id` | path | yes | integer (int64) | Telephony preset ID |
| `session_id` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`CreateOutboundCallDto`](#schema-createoutboundcalldto)

> Request to initiate an outbound call

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `fromNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Your outbound phone number to call from (must be in telephony preset) |
| `toNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Phone number to call |
| `playDialtone` | boolean | yes |  | Play dial tone to room until call is answered |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Created | [`OutboundCallResponseDto`](#schema-outboundcallresponsedto) |


#### ▸ Audio Service

_Endpoints for audio service access. Required header: 'X-Service-Key' (audio category). Optional: 'X-Service-Name'_

#### `POST /api/service/audio/audio-session/notification`

**audioSessionNotification**

- **Auth:** `service-key-audio` or `service-name`
- **operationId:** `audioSessionNotification`

**Request body** (`application/json`)

Schema: [`AudioSessionNotification`](#schema-audiosessionnotification)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `agentId` | string | yes |  |  |
| `sessionId` | string | yes |  |  |
| `participantIdentity` | string | yes |  |  |
| `type` | string | yes | `audio_session_started`, `audio_session_ended` |  |
| `stt_time_s` | number (double) | yes |  |  |
| `tts_char_count` | integer (int32) | yes |  |  |
| `mic_sample_rate` | integer (int32) |  |  |  |
| `mic_channels` | integer (int32) |  |  |  |
| `tts_sample_rate` | integer (int32) |  |  |  |
| `tts_voice_provider` | string | yes |  |  |
| `tts_voice_model_id` | string | yes |  |  |
| `stt_default_language` | string | yes |  |  |
| `total_call_duration_s` | number (double) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `POST /api/service/audio/inbound-session`

**initInboundSession**

- **Auth:** `service-key-audio` or `service-name`
- **operationId:** `initInboundSession`

**Request body** (`application/json`)

Schema: [`InboundSessionRequest`](#schema-inboundsessionrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `participantIdentity` | string | yes | minLength=`1` | Participant identity |
| `voicePresetReferenceId` | string | yes | minLength=`1` | Reference ID of the voice preset |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`InboundSessionResponse`](#schema-inboundsessionresponse) |


#### `DELETE /api/service/audio/rooms/{roomName}`

**deleteRoom**

- **Auth:** `service-key-audio` or `service-name`
- **operationId:** `deleteRoom`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `roomName` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `POST /api/service/audio/sip/transfer`

**transferSipParticipant**

- **Auth:** `service-key-audio` or `service-name`
- **operationId:** `transferSipParticipant`

**Request body** (`application/json`)

Schema: [`SipTransferRequestDto`](#schema-siptransferrequestdto)

> Request to transfer a SIP participant to another destination

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `participantIdentity` | string | yes | minLength=`1` | The ID of the participant to transfer |
| `roomName` | string | yes | minLength=`1` | The current room name |
| `transferTo` | string | yes | minLength=`1` | The destination (phone number with tel: prefix or SIP URI) |
| `playDialtone` | boolean | yes |  | Play dial tone during transfer (if false, room audio plays) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### ▸ Audio Agents v1

_Audio agent management and voice session APIs with API key authentication_

#### `GET /client/api/v1/audio-agents`

**List available audio agents**

Returns all audio agents (voice presets) available for the authenticated project with their configuration for WebRTC integration

- **Auth:** `api-key`
- **operationId:** `listVoicePresets`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Audio agents retrieved successfully | [`ApiResponseListVoicePresetClientDto`](#schema-apiresponselistvoicepresetclientdto) |
| `401` | Invalid API key | [`ApiResponseListVoicePresetClientDto`](#schema-apiresponselistvoicepresetclientdto) |
| `429` | Rate limit exceeded | [`ApiResponseListVoicePresetClientDto`](#schema-apiresponselistvoicepresetclientdto) |
| `500` | Internal server error | [`ApiResponseListVoicePresetClientDto`](#schema-apiresponselistvoicepresetclientdto) |


#### `POST /client/api/v1/audio-agents`

**Create a new audio agent**

Creates a new audio agent (voice preset) with the specified configuration

- **Auth:** `api-key`
- **operationId:** `createVoicePreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_id` | query |  | integer (int64) | Project numeric ID (deprecated, use project_ref_id) |
| `project_ref_id` | query |  | string | Project reference ID (e.g. proj_...) |

**Request body** (`application/json`)

Schema: [`VoicePresetRequestDto`](#schema-voicepresetrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Audio agent created successfully | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `400` | Invalid request parameters | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `401` | Invalid API key | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `429` | Rate limit exceeded | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `500` | Internal server error | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |


#### `GET /client/api/v1/audio-agents/mode`

**Get voice mode**

Returns the configured voice mode (LIVEKIT or WEBRTC)

- **Auth:** `api-key`
- **operationId:** `getVoiceMode`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Voice mode retrieved successfully | [`VoiceModeResponse`](#schema-voicemoderesponse) |
| `401` | Invalid API key | [`VoiceModeResponse`](#schema-voicemoderesponse) |


#### `GET /client/api/v1/audio-agents/options`

**Get voice preset options**

Returns a simplified list of voice presets for dropdowns (id, referenceId, name only)

- **Auth:** `api-key`
- **operationId:** `getOptions`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Options retrieved successfully | [`VoicePresetOptionDto`](#schema-voicepresetoptiondto)[] |
| `401` | Invalid API key | [`VoicePresetOptionDto`](#schema-voicepresetoptiondto)[] |
| `429` | Rate limit exceeded | [`VoicePresetOptionDto`](#schema-voicepresetoptiondto)[] |


#### `GET /client/api/v1/audio-agents/{idOrRef}`

**Get audio agent**

Retrieves detailed information about a specific audio agent by numeric ID (deprecated) or string reference ID (e.g. voice_...)

- **Auth:** `api-key`
- **operationId:** `getVoicePreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Audio agent retrieved successfully | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `401` | Invalid API key | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `404` | Audio agent not found | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `429` | Rate limit exceeded | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |


#### `PUT /client/api/v1/audio-agents/{idOrRef}`

**Update an audio agent**

Updates an existing audio agent's configuration by numeric ID (deprecated) or string reference ID (e.g. voice_...)

- **Auth:** `api-key`
- **operationId:** `updateVoicePreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Request body** (`application/json`)

Schema: [`VoicePresetRequestDto`](#schema-voicepresetrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Audio agent updated successfully | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `400` | Invalid request parameters | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `401` | Invalid API key | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `404` | Audio agent not found | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `429` | Rate limit exceeded | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |
| `500` | Internal server error | [`ApiResponseVoicePresetDto`](#schema-apiresponsevoicepresetdto) |


#### `DELETE /client/api/v1/audio-agents/{idOrRef}`

**Delete an audio agent**

Deletes an existing audio agent permanently by numeric ID (deprecated) or string reference ID (e.g. voice_...)

- **Auth:** `api-key`
- **operationId:** `deleteVoicePreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Audio agent deleted successfully |  |
| `401` | Invalid API key |  |
| `404` | Audio agent not found |  |
| `429` | Rate limit exceeded |  |


#### `POST /client/api/v1/audio-agents/{idOrRef}/start-session`

**Start voice session**

Creates a new voice session with the specified audio agent. Accepts numeric ID (deprecated) or string reference ID (e.g. voice_...).

- **Auth:** `api-key`
- **operationId:** `startVoiceSession`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |
| `user_id` | query |  | string | User identifier for session tracking |
| `session_id` | query |  | string | Session identifier for conversation continuity |

**Request body** (`application/json`)

Schema: [`VoiceSessionRequest`](#schema-voicesessionrequest)

> Optional Request to start a voice session

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `video` | [`VoiceVideoRequest`](#schema-voicevideorequest) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseVoiceSessionResponse`](#schema-apiresponsevoicesessionresponse) |


#### `GET /client/api/v1/audio-agents/{idOrRef}/stt/test-credentials`

**Test STT credentials**

Tests the Speech-to-Text credentials by making a test API call. Accepts numeric ID (deprecated) or string reference ID.

- **Auth:** `api-key`
- **operationId:** `testSttCredentials`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `GET /client/api/v1/audio-agents/{idOrRef}/tts/test-credentials`

**Test TTS credentials**

Tests the Text-to-Speech credentials by making a test API call. Accepts numeric ID (deprecated) or string reference ID.

- **Auth:** `api-key`
- **operationId:** `testTtsCredentials`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `GET /client/api/v1/audio-agents/{idOrRef}/voices`

**Get available voices for audio agent**

Returns the list of available voices for the specified audio agent's TTS provider. Accepts numeric ID (deprecated) or string reference ID.

- **Auth:** `api-key`
- **operationId:** `getVoices`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`Voice`](#schema-voice)[] |


#### `GET /client/api/v1/audio-agents/{idOrRef}/voices/{voiceId}/preview`

**Get voice preview**

Returns a presigned URL for a voice preview audio file. Accepts numeric ID (deprecated) or string reference ID.

- **Auth:** `api-key`
- **operationId:** `getVoicePreview_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Audio agent numeric ID or reference ID (e.g. voice_...) |
| `voiceId` | path | yes | string | Voice ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePreviewResponse`](#schema-voicepreviewresponse) |


---

<a id="dom-knowledge-storage-embeddings"></a>

### Knowledge, Storage & Embeddings

_15 operations._

#### ▸ Storage Resources

_Manage vector storage resources_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources`

**List storage resources**

Lists all storage resources for the project

- **Auth:** `bearer-jwt`
- **operationId:** `listStorageResources_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resources retrieved successfully | [`StorageResourceResponse`](#schema-storageresourceresponse)[] |
| `403` | User does not have access to project | [`StorageResourceResponse`](#schema-storageresourceresponse)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/storage-resources`

**Create storage resource**

Create a new vector store with a Weaviate collection.

Only **name**, **llmProviderRefId** (or deprecated **llmConfigId**), and **embeddingModelId** are required. All other fields are optional:

- **metadataSchema** *(optional)* — Typed property definitions for metadata filtering.
- **invertedIndexConfig** *(optional)* — BM25 and keyword search tuning. bm25_b and bm25_k1 must be set together. [https://docs.weaviate.io/weaviate/config-refs/indexing/inverted-index](https://docs.weaviate.io/weaviate/config-refs/indexing/inverted-index)
- **vectorIndexType** *(optional)* — HNSW (default), FLAT, or DYNAMIC.
- **vectorIndexConfig** *(optional)* — Vector index tuning. Available keys depend on vectorIndexType. [https://docs.weaviate.io/weaviate/config-refs/indexing/vector-index](https://docs.weaviate.io/weaviate/config-refs/indexing/vector-index)
- **searchConfig** *(optional)* — Search behavior: search_type (vector, keyword, or hybrid) and hybrid_search_alpha (0.0–1.0, must be 0.0 for keyword).

- **Auth:** `bearer-jwt`
- **operationId:** `createStorageResource_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`StorageResourceCreateRequest`](#schema-storageresourcecreaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` | Name of the storage resource |
| `llmProviderRefId` | string |  |  | Reference ID (UUID) of the LLM provider configuration to use for embeddings. Obtain from GET /storage-resources/llm-providers (providerRefId field). Preferred over llmConfigId. |
| `llmConfigId` | integer (int64) |  |  | Internal numeric ID of the LLM provider configuration. Deprecated — use llmProviderRefId instead. |
| `embeddingModelId` | string | yes | minLength=`1` | ID of the embedding model to use |
| `metadataSchema` | [`MetadataSchemaField`](#schema-metadataschemafield)[] |  |  | Schema definition for metadata fields used to filter documents during search/retrieval |
| `invertedIndexConfig` | object |  |  | BM25 and inverted index tuning. bm25_b and bm25_k1 must be set together. See https://docs.weaviate.io/weaviate/config-refs/indexing/inverted-index |
| `vectorIndexType` | string |  | `hnsw`, `flat`, `dynamic`, `HNSW`, `FLAT`, `DYNAMIC` | Vector index type: hnsw (default), flat, or dynamic. |
| `vectorIndexConfig` | object |  |  | Vector index tuning. Available keys depend on vector_index_type (HNSW, FLAT, DYNAMIC). See https://docs.weaviate.io/weaviate/config-refs/indexing/vector-index |
| `searchConfig` | object |  |  | Search behavior configuration. Allows setting the search type (vector, keyword, or hybrid) and hybrid_search_alpha (0.0–1.0, must be 0.0 for keyword). |
| `isLlmProviderSpecified` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resource created successfully | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `403` | User does not have access to project | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `503` | Storage service unavailable | [`StorageResourceResponse`](#schema-storageresourceresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/chunking-strategies`

**Get chunking strategies**

Lists all available chunking strategies with their descriptions and default parameters

- **Auth:** `bearer-jwt`
- **operationId:** `getChunkingStrategies`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Chunking strategies retrieved successfully | [`ChunkingStrategiesResponse`](#schema-chunkingstrategiesresponse) |
| `403` | User does not have access to organization | [`ChunkingStrategiesResponse`](#schema-chunkingstrategiesresponse) |
| `503` | Storage service unavailable | [`ChunkingStrategiesResponse`](#schema-chunkingstrategiesresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/llm-providers`

**Get configured embedding model providers**

Lists LLM provider configurations with available embedding models for vector stores

- **Auth:** `bearer-jwt`
- **operationId:** `getConfiguredLlmProviders_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Embedding providers retrieved successfully | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |
| `403` | User does not have access to organization | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}`

**Get storage resource**

Retrieves a single storage resource by ID

- **Auth:** `bearer-jwt`
- **operationId:** `getStorageResource_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | string (uuid) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resource retrieved successfully | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `403` | User does not have access to this resource | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `404` | Storage resource not found | [`StorageResourceResponse`](#schema-storageresourceresponse) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}`

**Update storage resource**

Updates the name of a storage resource

- **Auth:** `bearer-jwt`
- **operationId:** `updateStorageResource_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | string (uuid) |  |

**Request body** (`application/json`)

Schema: [`StorageResourceUpdateRequest`](#schema-storageresourceupdaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resource updated successfully | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `403` | User does not have access to this resource | [`StorageResourceResponse`](#schema-storageresourceresponse) |
| `404` | Storage resource not found | [`StorageResourceResponse`](#schema-storageresourceresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}`

**Delete storage resource**

Deletes a storage resource

- **Auth:** `bearer-jwt`
- **operationId:** `deleteStorageResource_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | string (uuid) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Storage resource deleted successfully |  |
| `403` | User does not have access to this resource |  |
| `404` | Storage resource not found |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/crawl`

**Crawl URLs**

Initiates web crawling for one or more URLs and adds content to the vector store

- **Auth:** `bearer-jwt`
- **operationId:** `crawlUrl_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |

**Request body** (`application/json`)

Schema: [`CrawlRequest`](#schema-crawlrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `urls` | [`CrawlUrlRequest`](#schema-crawlurlrequest)[] | yes |  |  |
| `ignoreLinks` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Crawl task initiated successfully | [`CrawlResultDto`](#schema-crawlresultdto) |
| `403` | User does not have access to this resource | [`CrawlResultDto`](#schema-crawlresultdto) |
| `404` | Storage resource not found | [`CrawlResultDto`](#schema-crawlresultdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files`

**List files**

Lists files in the vector store with pagination

- **Auth:** `bearer-jwt`
- **operationId:** `listFiles_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `page` | query |  | integer (int32) |  |
| `pageSize` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Files retrieved successfully | [`FileListDto`](#schema-filelistdto) |
| `403` | User does not have access to this resource | [`FileListDto`](#schema-filelistdto) |
| `404` | Storage resource not found | [`FileListDto`](#schema-filelistdto) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files`

**Upload files to storage**

Uploads files to the vector store. Files are processed asynchronously for embedding. Optionally specify chunking strategy, configuration, and metadata for filtering.

- **Auth:** `bearer-jwt`
- **operationId:** `uploadFiles_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `files` | query | yes | string (binary)[] |  |
| `chunkingStrategy` | query |  | string |  |
| `chunkingConfig` | query |  | string |  |
| `metadata` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | File upload job started successfully | [`FileUploadResultDto`](#schema-fileuploadresultdto) |
| `400` | Invalid chunking configuration or metadata JSON | [`FileUploadResultDto`](#schema-fileuploadresultdto) |
| `403` | User does not have access to this resource | [`FileUploadResultDto`](#schema-fileuploadresultdto) |
| `404` | Storage resource not found | [`FileUploadResultDto`](#schema-fileuploadresultdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}`

**Delete file**

Deletes a file from the vector store

- **Auth:** `bearer-jwt`
- **operationId:** `deleteFile_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `fileId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | File deleted successfully |  |
| `403` | User does not have access to this resource |  |
| `404` | Storage resource or file not found |  |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}/download`

**Download file**

Downloads the original file from the storage resource

- **Auth:** `bearer-jwt`
- **operationId:** `downloadFile_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `fileId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | File downloaded successfully | `string` |
| `404` | Storage resource or file not found | `string` |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}/parsed-text/download`

**Download parsed text**

Downloads the parsed text content from a file in the storage resource

- **Auth:** `bearer-jwt`
- **operationId:** `downloadParsedText`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `fileId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Parsed text downloaded successfully | `string` |
| `404` | Storage resource or file not found | `string` |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/search`

**Search vector store**

Performs semantic search on the vector store

- **Auth:** `bearer-jwt`
- **operationId:** `search`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `vectorStoreId` | path | yes | string (uuid) |  |
| `query` | query | yes | string |  |
| `numberOfDocuments` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Search completed successfully | [`SearchResponse`](#schema-searchresponse) |
| `403` | User does not have access to this resource | [`SearchResponse`](#schema-searchresponse) |
| `404` | Storage resource not found | [`SearchResponse`](#schema-searchresponse) |


#### ▸ Embeddings v1

_OpenAI-compatible embeddings proxy_

#### `POST /client/api/v1/embeddings`

**Generate embeddings (OpenAI-compatible proxy)**

- **Auth:** `api-key`
- **operationId:** `createEmbedding`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `provider_ref_id` | query | yes | string (uuid) |  |

**Request body** (`application/json`)

Schema: [`EmbeddingsRequest`](#schema-embeddingsrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `model` | string | yes |  |  |
| `input` | string | yes |  |  |
| `encoding_format` | string |  |  |  |
| `dimensions` | integer (int32) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


---

<a id="dom-tools-external-integrations"></a>

### Tools & External Integrations

_33 operations._

#### ▸ MCP Tools

_Unified MCP tools management_

#### `GET /api/mcp-tools`

**List all available MCP tools**

Fetches all available MCP tools from the AI service

- **Auth:** `bearer-jwt`
- **operationId:** `getMCPTools`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successfully retrieved MCP tools | [`MCPTool`](#schema-mcptool)[] |
| `403` | Forbidden - insufficient permissions | [`AiToolInfo`](#schema-aitoolinfo)[] |
| `503` | Service unavailable - AI service is unreachable | [`AiToolInfo`](#schema-aitoolinfo)[] |


#### `POST /api/mcp-tools/invoke`

**Invoke MCP Tool**

Executes an MCP tool with the provided parameters. The request should contain the tool name, MCP server ID, and any required parameters for the tool.

- **Auth:** `bearer-jwt`
- **operationId:** `invokeMCPTool`

**Request body** (`application/json`)

```json
{
  "type": "object",
  "additionalProperties": {}
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successfully invoked MCP tool - returns tool execution result | `object` |
| `400` | Bad request - invalid tool name, server ID, or parameters | `object` |
| `403` | Forbidden - insufficient permissions | `object` |
| `404` | Not found - tool or MCP server not found | `object` |
| `503` | Service unavailable - AI service is unreachable | `object` |


#### ▸ n8n Deployment Configuration

_Configuration and SSH key management for n8n deployment_

#### `GET /api/n8n-deployment/domain`

**Get Route53 domain configuration**

Retrieves the configured Route53 domain that will be used for n8n deployment subdomains

- **Auth:** `bearer-jwt`
- **operationId:** `getRoute53Domain`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Domain retrieved successfully | [`N8nDomainResponse`](#schema-n8ndomainresponse) |
| `500` | Internal server error | [`N8nDomainResponse`](#schema-n8ndomainresponse) |


#### `GET /api/n8n-deployment/ssh-public-key`

**Get SSH public key for deployment**

Retrieves the SSH public key that must be added to the target server's authorized_keys file to allow the deployment service to connect. Returns the key along with instructions and a command for easy setup.

- **Auth:** `bearer-jwt`
- **operationId:** `getSSHPublicKey`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | SSH public key retrieved successfully | [`SSHPublicKeyResponse`](#schema-sshpublickeyresponse) |
| `500` | Internal server error | [`SSHPublicKeyResponse`](#schema-sshpublickeyresponse) |
| `503` | Deployment service unavailable | [`SSHPublicKeyResponse`](#schema-sshpublickeyresponse) |


#### ▸ n8n Deployment

_n8n deployment management endpoints_

#### `GET /api/organizations/{organizationId}/n8n-deployment`

**Get organization's n8n deployment**

Retrieves the n8n deployment details for an organization. Organizations can only have one n8n deployment at a time. Returns 404 if the organization does not have a deployment yet.

- **Auth:** `bearer-jwt`
- **operationId:** `getOrganizationDeployment`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Deployment found and returned | [`OrganizationDeploymentResponse`](#schema-organizationdeploymentresponse) |
| `403` | User not authorized for this organization | [`OrganizationDeploymentResponse`](#schema-organizationdeploymentresponse) |
| `404` | Organization not found or has no deployment | [`OrganizationDeploymentResponse`](#schema-organizationdeploymentresponse) |


#### `POST /api/organizations/{organizationId}/n8n-deployment`

**Initiate n8n deployment**

Creates a new n8n deployment for the organization with the specified configuration. Validates subdomain availability, stores credentials securely in Vault, and initiates the deployment process.

- **Auth:** `bearer-jwt`
- **operationId:** `initiateDeployment`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`InitiateDeploymentRequest`](#schema-initiatedeploymentrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `subdomain` | string |  | minLength=`0` ; maxLength=`63` ; pattern=`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` |  |
| `serverIp` | string | yes | minLength=`1` ; pattern=`^((25[0-5]|(2[0-4]|1\d|[1-9]|)\d)\.?\b){4}$` |  |
| `sshUsername` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `n8nUsername` | string (email) | yes | minLength=`1` |  |
| `n8nPassword` | string | yes | minLength=`12` ; maxLength=`2147483647` |  |
| `port` | integer (int32) | yes | minimum=`1` ; maximum=`65535` |  |
| `workersCount` | integer (int32) | yes | minimum=`1` ; maximum=`4` |  |
| `customUrl` | string |  | minLength=`0` ; maxLength=`500` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Deployment initiated successfully | [`DeploymentResponse`](#schema-deploymentresponse) |
| `400` | Invalid request or subdomain unavailable | [`DeploymentResponse`](#schema-deploymentresponse) |
| `403` | User not authorized for this organization | [`DeploymentResponse`](#schema-deploymentresponse) |
| `404` | Organization not found | [`DeploymentResponse`](#schema-deploymentresponse) |
| `409` | Organization already has a deployment | [`DeploymentResponse`](#schema-deploymentresponse) |


#### `DELETE /api/organizations/{organizationId}/n8n-deployment`

**Delete organization's n8n deployment**

Deletes the n8n deployment for an organization. This removes the deployment record and cleans up associated resources (Vault password). Note: DNS records are not automatically deleted and must be handled separately.

- **Auth:** `bearer-jwt`
- **operationId:** `deleteDeployment`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Deployment deleted successfully |  |
| `403` | User not authorized for this organization |  |
| `404` | Organization not found or has no deployment |  |


#### `POST /api/organizations/{organizationId}/n8n-deployment/cancel`

**Cancel an in-progress deployment**

Cancels a deployment that is currently in PENDING, DNS_REGISTERING, SSH_CONNECTING, or DEPLOYING state. The deployment worker is interrupted and /opt/n8n is cleaned up on the remote server.

- **Auth:** `bearer-jwt`
- **operationId:** `cancelDeployment`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `202` | Cancellation initiated | [`CancelDeploymentResponse`](#schema-canceldeploymentresponse) |
| `403` | User not authorized for this organization | [`CancelDeploymentResponse`](#schema-canceldeploymentresponse) |
| `404` | Organization not found or has no deployment | [`CancelDeploymentResponse`](#schema-canceldeploymentresponse) |
| `409` | Deployment is not in a cancellable state | [`CancelDeploymentResponse`](#schema-canceldeploymentresponse) |


#### `GET /api/organizations/{organizationId}/n8n-deployment/logs`

**Get deployment logs**

Retrieves paginated deployment logs with optional log level filtering.

- **Auth:** `bearer-jwt`
- **operationId:** `getDeploymentLogs`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `logLevel` | query |  | string | Filter by log level (INFO, WARN, ERROR, DEBUG) |
| `page` | query |  | integer (int32) | Page number (0-indexed) |
| `size` | query |  | integer (int32) | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Logs retrieved successfully | [`PageDeploymentLogDto`](#schema-pagedeploymentlogdto) |
| `403` | User not authorized for this organization | [`PageDeploymentLogDto`](#schema-pagedeploymentlogdto) |
| `404` | Organization or deployment not found | [`PageDeploymentLogDto`](#schema-pagedeploymentlogdto) |


#### `GET /api/organizations/{organizationId}/n8n-deployment/status`

**Get deployment status**

Retrieves the current status and details of the organization's n8n deployment

- **Auth:** `bearer-jwt`
- **operationId:** `getDeploymentStatus`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Deployment status retrieved successfully | [`DeploymentStatusResponse`](#schema-deploymentstatusresponse) |
| `403` | User not authorized for this organization | [`DeploymentStatusResponse`](#schema-deploymentstatusresponse) |
| `404` | Organization or deployment not found | [`DeploymentStatusResponse`](#schema-deploymentstatusresponse) |


#### `POST /api/organizations/{organizationId}/n8n-deployment/validate-subdomain`

**Validate subdomain availability**

Checks if a subdomain is available for n8n deployment by validating format and checking against existing deployments and DNS records

- **Auth:** `bearer-jwt`
- **operationId:** `validateSubdomain`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`ValidateSubdomainRequest`](#schema-validatesubdomainrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `subdomain` | string | yes | minLength=`3` ; maxLength=`63` ; pattern=`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Validation completed successfully | [`SubdomainValidationResponse`](#schema-subdomainvalidationresponse) |
| `400` | Invalid request body or subdomain format | [`SubdomainValidationResponse`](#schema-subdomainvalidationresponse) |
| `403` | User not authorized for this organization | [`SubdomainValidationResponse`](#schema-subdomainvalidationresponse) |
| `404` | Organization not found | [`SubdomainValidationResponse`](#schema-subdomainvalidationresponse) |


#### ▸ HTTP Requests

_Manage HTTP request configurations for AI agents_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/http-requests`

**List HTTP requests**

List all HTTP requests for the specified project with pagination

- **Auth:** `bearer-jwt`
- **operationId:** `listHttpRequests_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `page` | query |  | integer (int32) |  |
| `size` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP requests retrieved successfully | [`Page`](#schema-page) |
| `401` | Unauthorized - missing or invalid JWT token | [`PageHttpRequestResponse`](#schema-pagehttprequestresponse) |
| `403` | Forbidden - insufficient permissions | [`PageHttpRequestResponse`](#schema-pagehttprequestresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/http-requests`

**Create HTTP request**

Create a new HTTP request configuration for use by AI agents

- **Auth:** `bearer-jwt`
- **operationId:** `createHttpRequest_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`HttpRequestCreateRequest`](#schema-httprequestcreaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object | yes |  |  |
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object | yes |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | HTTP request created successfully | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `400` | Invalid input or validation failed | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `401` | Unauthorized - missing or invalid JWT token | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `403` | Forbidden - insufficient permissions | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `404` | Organization or project not found | [`HttpRequestResponse`](#schema-httprequestresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/http-requests/test`

**Test HTTP request**

Test an HTTP request configuration to validate endpoint connectivity

- **Auth:** `bearer-jwt`
- **operationId:** `testHttpRequest_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`HttpRequestTestData`](#schema-httprequesttestdata)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes |  |  |
| `headers` | object | yes |  |  |
| `data` | object |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request test completed (success or failure) | [`HttpRequestTestResponse`](#schema-httprequesttestresponse) |
| `400` | Invalid input or validation failed | [`HttpRequestTestResponse`](#schema-httprequesttestresponse) |
| `401` | Unauthorized - missing or invalid JWT token | [`HttpRequestTestResponse`](#schema-httprequesttestresponse) |
| `403` | Forbidden - insufficient permissions | [`HttpRequestTestResponse`](#schema-httprequesttestresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}`

**Get HTTP request**

Retrieve a specific HTTP request configuration by ID

- **Auth:** `bearer-jwt`
- **operationId:** `getHttpRequest_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request retrieved successfully | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `401` | Unauthorized - missing or invalid JWT token | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `403` | Forbidden - insufficient permissions | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `404` | HTTP request not found | [`HttpRequestResponse`](#schema-httprequestresponse) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}`

**Update HTTP request**

Update an existing HTTP request configuration

- **Auth:** `bearer-jwt`
- **operationId:** `updateHttpRequest_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`HttpRequestUpdateRequest`](#schema-httprequestupdaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object |  |  |  |
| `httpMethod` | string |  | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string |  | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object |  |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request updated successfully | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `400` | Invalid input or validation failed | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `401` | Unauthorized - missing or invalid JWT token | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `403` | Forbidden - insufficient permissions | [`HttpRequestResponse`](#schema-httprequestresponse) |
| `404` | HTTP request not found | [`HttpRequestResponse`](#schema-httprequestresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}`

**Delete HTTP request**

Soft delete an HTTP request configuration. Cannot delete if agents reference it

- **Auth:** `bearer-jwt`
- **operationId:** `deleteHttpRequest_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | HTTP request deleted successfully |  |
| `401` | Unauthorized - missing or invalid JWT token |  |
| `403` | Forbidden - insufficient permissions |  |
| `404` | HTTP request not found |  |
| `409` | Conflict - HTTP request is referenced by one or more agents |  |


#### ▸ MCP Server Management

_MCP server CRUD operations_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/mcp-servers`

**List MCP servers for a project**

Retrieves all MCP servers belonging to a specific project, enriched with health data from AI service

- **Auth:** `bearer-jwt`
- **operationId:** `listMCPServers`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | MCP servers retrieved successfully | [`MCPServerResponse`](#schema-mcpserverresponse)[] |
| `403` | User does not have permission to view MCP servers in this project | [`MCPServerResponse`](#schema-mcpserverresponse)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/mcp-servers`

**Create a new MCP server**

Creates a new MCP server with specified configuration. The server is created both in the database and in the AI service.

- **Auth:** `bearer-jwt`
- **operationId:** `createMCPServer`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |

**Request body** (`application/json`)

Schema: [`MCPServerCreateRequest`](#schema-mcpservercreaterequest)

> Request to create a new MCP server

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string | yes | `stdio`, `streamable-http`, `sse` | Transport type. Use 'stdio' for process-based MCP servers (requires mcp-proxy), 'streamable-http' for HTTP servers, or 'sse' for Server-Sent Events |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports (required for streamable-http and sse). Not used for stdio transport. |
| `baseCommand` | string |  |  | Base command for stdio transport (required for stdio). Format: 'command arg1 arg2 ...'. Supports quoted arguments with spaces (both double and single quotes). Example: 'echo "hello world" test' will parse as command='echo' args=['hello world', 'test'] |
| `defaultEnvVars` | object |  |  | Default environment variables for stdio transport |
| `defaultTimeoutSeconds` | integer (int32) | yes |  | Default timeout in seconds |
| `authType` | string | yes | `none`, `bearer`, `api_key` ; minLength=`1` | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean | yes |  | Whether the server is enabled |
| `metadataSchema` | object |  |  | JSON Schema defining metadata structure for tools. Must have 'type' and 'properties' fields. |
| `parameterMappings` | [`ParameterMapping`](#schema-parametermapping)[] |  |  | Parameter mappings for MCP tools |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | MCP server created successfully | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `400` | Invalid request data | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `403` | User does not have permission to create MCP servers in this project | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `404` | Organization or project not found | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `503` | AI service unavailable | [`MCPServerResponse`](#schema-mcpserverresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}`

**Get MCP server by UUID**

Retrieves MCP server details by AI service UUID

- **Auth:** `bearer-jwt`
- **operationId:** `getMCPServer`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | AI service MCP server UUID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | MCP server found | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `403` | User does not have permission to view this MCP server | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `404` | MCP server not found | [`MCPServerResponse`](#schema-mcpserverresponse) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}`

**Update an MCP server**

Updates an existing MCP server's configuration

- **Auth:** `bearer-jwt`
- **operationId:** `updateMCPServer`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | AI service MCP server UUID |

**Request body** (`application/json`)

Schema: [`MCPServerUpdateRequest`](#schema-mcpserverupdaterequest)

> Request to update an existing MCP server

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`1` ; maxLength=`255` | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string |  | `stdio`, `streamable-http`, `sse` | Transport type. Use 'stdio' for process-based MCP servers (requires mcp-proxy), 'streamable-http' for HTTP servers, or 'sse' for Server-Sent Events |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports (required for streamable-http and sse). Not used for stdio transport. |
| `baseCommand` | string |  |  | Base command for stdio transport (required for stdio). Format: 'command arg1 arg2 ...'. Supports quoted arguments with spaces (both double and single quotes). Updating this triggers mcp-proxy re-orchestration. |
| `defaultEnvVars` | object |  |  | Default environment variables for stdio transport |
| `defaultTimeoutSeconds` | integer (int32) |  |  | Default timeout in seconds |
| `authType` | string |  | `none`, `bearer`, `api_key` | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean |  |  | Whether the server is enabled |
| `metadataSchema` | object |  |  | JSON Schema defining metadata structure for tools. Must have 'type' and 'properties' fields. |
| `parameterMappings` | [`ParameterMapping`](#schema-parametermapping)[] |  |  | Parameter mappings for MCP tools |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | MCP server updated successfully | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `400` | Invalid request data | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `403` | User does not have permission to update this MCP server | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `404` | MCP server not found | [`MCPServerResponse`](#schema-mcpserverresponse) |
| `503` | AI service unavailable | [`MCPServerResponse`](#schema-mcpserverresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}`

**Delete an MCP server**

Soft deletes an MCP server. The server is marked as deleted in the database and deletion is attempted in the AI service (best effort).

- **Auth:** `bearer-jwt`
- **operationId:** `deleteMCPServer`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | AI service MCP server UUID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | MCP server deleted successfully |  |
| `403` | User does not have permission to delete this MCP server |  |
| `404` | MCP server not found |  |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/health`

**Get MCP server health status**

Retrieves the health status of an MCP server

- **Auth:** `bearer-jwt`
- **operationId:** `getMCPServerHealth`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | MCP server ID |
| `force_refresh` | query |  | boolean | Force refresh health status from the source (bypass cache) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | MCP server health status retrieved | [`MCPHealthResponse`](#schema-mcphealthresponse) |
| `404` | MCP server not found | [`MCPHealthResponse`](#schema-mcphealthresponse) |
| `403` | No access rights to the organization | [`MCPHealthResponse`](#schema-mcphealthresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/test`

**Test MCP server**

Tests connectivity and functionality of an MCP server

- **Auth:** `bearer-jwt`
- **operationId:** `testMCPServer`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | MCP server UUID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | MCP server test completed | `object` |
| `404` | MCP server not found | `object` |
| `403` | No access rights to the organization | `object` |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/tools`

**List available tools for an MCP server**

Returns a list of available tools for the specified MCP server

- **Auth:** `bearer-jwt`
- **operationId:** `getMCPServerTools`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | AI service MCP server UUID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Tools retrieved successfully | [`MCPServerToolsResponse`](#schema-mcpservertoolsresponse) |
| `403` | User does not have permission to view this MCP server | [`MCPServerToolsResponse`](#schema-mcpservertoolsresponse) |
| `404` | MCP server not found | [`MCPServerToolsResponse`](#schema-mcpservertoolsresponse) |
| `503` | AI service unavailable | [`MCPServerToolsResponse`](#schema-mcpservertoolsresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/tools/{toolName}/call`

**Call a specific tool on an MCP server**

Executes the specified tool on the given MCP server within the context of the organization and project.

- **Auth:** `bearer-jwt`
- **operationId:** `callMCPServerTool`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | path | yes | integer (int64) | Project ID |
| `serverId` | path | yes | string (uuid) | MCP server UUID |
| `toolName` | path | yes | string | Tool name |

**Request body** (`application/json`)

```json
{
  "type": "object",
  "additionalProperties": {}
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Tool executed successfully | `object` |
| `400` | Invalid request or tool execution error | `object` |
| `403` | Access to the organization is forbidden | `object` |
| `404` | Tool or MCP server not found | `object` |


#### ▸ Skill Management

_CRUD and execution for AI service skills_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/skills`

**List skills for a project**

Retrieves all skills belonging to a specific project with pagination and filtering support. Use query parameters: page (0-indexed), size (default 20), sort (e.g. 'name,asc' or 'createdAt,desc'), name (optional filter by name, case-insensitive partial match)

- **Auth:** `bearer-jwt`
- **operationId:** `listSkills_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `name` | query |  | string |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skills retrieved successfully | [`PageSkillResponse`](#schema-pageskillresponse) |
| `403` | Insufficient permissions | [`PageSkillResponse`](#schema-pageskillresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/skills`

**Create a new skill**

- **Auth:** `bearer-jwt`
- **operationId:** `createSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`SkillCreateRequest`](#schema-skillcreaterequest)

> Request to create a new skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`2` ; maxLength=`64` ; pattern=`^[a-z0-9][a-z0-9-]*[a-z0-9]$` |  |
| `description` | string | yes | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Skill created successfully | [`SkillResponse`](#schema-skillresponse) |
| `400` | Invalid request data | [`SkillResponse`](#schema-skillresponse) |
| `404` | Organization or project not found | [`SkillResponse`](#schema-skillresponse) |
| `503` | AI service unavailable | [`SkillResponse`](#schema-skillresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}`

**Get a skill by ID**

- **Auth:** `bearer-jwt`
- **operationId:** `getSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `skillId` | path | yes | string | AI service skill ID (UUID) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill found | [`SkillResponse`](#schema-skillresponse) |
| `404` | Skill not found | [`SkillResponse`](#schema-skillresponse) |


#### `PATCH /api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}`

**Partially update a skill**

- **Auth:** `bearer-jwt`
- **operationId:** `updateSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `skillId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`SkillUpdateRequest`](#schema-skillupdaterequest)

> Request to partially update an existing skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string |  | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |
| `isActive` | boolean |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill updated successfully | [`SkillResponse`](#schema-skillresponse) |
| `404` | Skill not found | [`SkillResponse`](#schema-skillresponse) |
| `503` | AI service unavailable | [`SkillResponse`](#schema-skillresponse) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}`

**Soft-delete a skill**

- **Auth:** `bearer-jwt`
- **operationId:** `deleteSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `skillId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Skill deleted successfully |  |
| `404` | Skill not found |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}/activate`

**Reactivate a soft-deleted skill**

- **Auth:** `bearer-jwt`
- **operationId:** `activateSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `skillId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill activated successfully | [`SkillResponse`](#schema-skillresponse) |
| `404` | Skill not found | [`SkillResponse`](#schema-skillresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}/execute`

**Execute a skill script in a sandbox**

- **Auth:** `bearer-jwt`
- **operationId:** `executeSkill_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `skillId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`SkillExecuteRequest`](#schema-skillexecuterequest)

> Request to execute a skill script

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `scriptName` | string | yes | minLength=`1` |  |
| `args` | string[] |  |  |  |
| `timeout` | integer (int32) |  |  | Execution timeout in seconds (1–600) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Script executed successfully | [`SkillExecuteResponse`](#schema-skillexecuteresponse) |
| `400` | Script not found in skill | [`SkillExecuteResponse`](#schema-skillexecuteresponse) |
| `404` | Skill not found | [`SkillExecuteResponse`](#schema-skillexecuteresponse) |
| `503` | Sandbox execution service unreachable | [`SkillExecuteResponse`](#schema-skillexecuteresponse) |


---

<a id="dom-observability-traces-evaluations-usage"></a>

### Observability: Traces, Evaluations & Usage

_18 operations._

#### ▸ Evaluations

_Unified evaluation runs API for agents and teams with JWT authentication_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/eval-runs`

**List evaluation runs**

Returns a paginated list of evaluation runs.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Evaluation runs for a specific agent
            - `teamId`: Evaluation runs for a specific team

- **Auth:** `bearer-jwt`
- **operationId:** `listEvalRuns_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `modelId` | query |  | string |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |
| `evalTypes` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval runs retrieved successfully | [`PageEvalRunListDto`](#schema-pageevalrunlistdto) |
| `400` | Missing or multiple component filters | [`PageEvalRunListDto`](#schema-pageevalrunlistdto) |
| `401` | Unauthorized | [`PageEvalRunListDto`](#schema-pageevalrunlistdto) |
| `404` | Agent/Team not found | [`PageEvalRunListDto`](#schema-pageevalrunlistdto) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/eval-runs`

**Execute evaluation**

Runs an evaluation against an agent or team.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Run evaluation against this agent
            - `teamId`: Run evaluation against this team

- **Auth:** `bearer-jwt`
- **operationId:** `executeEval_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`EvalRunInputRequest`](#schema-evalruninputrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `eval_type` | string | yes |  |  |
| `input` | string | yes |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `model_id` | string |  |  |  |
| `model_provider` | string |  |  |  |
| `provider_ref_id` | string |  |  |  |
| `name` | string |  |  |  |
| `expected_output` | string |  |  |  |
| `criteria` | string |  |  |  |
| `scoring_strategy` | string |  |  |  |
| `threshold` | integer (int32) |  |  |  |
| `num_iterations` | integer (int32) | yes |  |  |
| `warmup_runs` | integer (int32) | yes |  |  |
| `additional_guidelines` | string |  |  |  |
| `additional_context` | string |  |  |  |
| `expected_tool_calls` | string[] |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Evaluation executed successfully | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `400` | Missing or multiple component filters | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `404` | Agent/Team not found | [`EvalRunDetailDto`](#schema-evalrundetaildto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}`

**Get evaluation run**

Retrieves detailed information about a specific evaluation run.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Eval run belongs to this agent
            - `teamId`: Eval run belongs to this team

- **Auth:** `bearer-jwt`
- **operationId:** `getEvalRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval run retrieved successfully | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `400` | Missing or multiple component filters | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `404` | Eval run not found | [`EvalRunDetailDto`](#schema-evalrundetaildto) |


#### `PATCH /api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}`

**Update evaluation run**

Updates an existing evaluation run.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Eval run belongs to this agent
            - `teamId`: Eval run belongs to this team

- **Auth:** `bearer-jwt`
- **operationId:** `updateEvalRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`UpdateEvalRunRequest`](#schema-updateevalrunrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  |  |  |
| `eval_data` | object |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval run updated successfully | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `400` | Missing or multiple component filters | [`EvalRunDetailDto`](#schema-evalrundetaildto) |
| `404` | Eval run not found | [`EvalRunDetailDto`](#schema-evalrundetaildto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}`

**Delete evaluation run**

Deletes an evaluation run by ID.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Eval run belongs to this agent
            - `teamId`: Eval run belongs to this team

- **Auth:** `bearer-jwt`
- **operationId:** `deleteEvalRun_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Eval run deleted successfully |  |
| `400` | Missing or multiple component filters |  |
| `404` | Eval run not found |  |


#### ▸ Playground

_Play with AI endpoints_

#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/run-stream`

**runAgentCreatorMessageStream**

- **Auth:** `bearer-jwt`
- **operationId:** `runAgentCreatorMessageStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agent_id` | query | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventMapStringObject`](#schema-serversenteventmapstringobject)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/runs`

**runAgentCreatorMessage**

- **Auth:** `bearer-jwt`
- **operationId:** `runAgentCreatorMessage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agent_id` | query | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/sessions/{sessionId}/runs`

**getAgentCreatorSessionRuns**

- **Auth:** `bearer-jwt`
- **operationId:** `getAgentCreatorSessionRuns`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `sessionId` | path | yes | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SessionRunDto`](#schema-sessionrundto)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/agents/{agentId}/run-stream`

**runAgentMessageStream**

- **Auth:** `bearer-jwt`
- **operationId:** `runAgentMessageStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |
| `tool_context` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventMapStringObject`](#schema-serversenteventmapstringobject)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/agents/{agentId}/runs`

**runAgentMessage**

- **Auth:** `bearer-jwt`
- **operationId:** `runAgentMessage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |
| `tool_context` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PlaygroundRunResponse`](#schema-playgroundrunresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/playground/config/limits`

**getUploadLimits**

- **Auth:** `bearer-jwt`
- **operationId:** `getUploadLimits`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`MultipartLimitsDto`](#schema-multipartlimitsdto) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/teams/{teamId}/run-stream`

**runTeamMessageStream**

- **Auth:** `bearer-jwt`
- **operationId:** `runTeamMessageStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventMapStringObject`](#schema-serversenteventmapstringobject)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/teams/{teamId}/runs`

**runTeamMessage**

- **Auth:** `bearer-jwt`
- **operationId:** `runTeamMessage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `teamId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `session_id` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PlaygroundRunResponse`](#schema-playgroundrunresponse) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/workflows/{workflowId}/run-stream`

**runWorkflowMessageStream**

- **Auth:** `bearer-jwt`
- **operationId:** `runWorkflowMessageStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `session_id` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ServerSentEventMapStringObject`](#schema-serversenteventmapstringobject)[] |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/playground/workflows/{workflowId}/runs`

**runWorkflowMessage**

- **Auth:** `bearer-jwt`
- **operationId:** `runWorkflowMessage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `workflowId` | path | yes | string |  |
| `session_id` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### ▸ Traces

_Unified trace observability API for agents and teams with JWT authentication_

#### `GET /api/organizations/{organizationId}/projects/{projectId}/traces`

**List traces**

Returns a paginated list of execution traces.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Filter traces for a specific agent
            - `teamId`: Filter traces for a specific team

            **Optional Filters**: userId, run_id, session_id, status, start_time, end_time

- **Auth:** `bearer-jwt`
- **operationId:** `listTraces_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `userId` | query |  | string |  |
| `runId` | query |  | string |  |
| `sessionId` | query |  | string |  |
| `status` | query |  | string |  |
| `startTime` | query |  | string |  |
| `endTime` | query |  | string |  |
| `page` | query |  | integer (int32) |  |
| `limit` | query |  | integer (int32) |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Traces retrieved successfully | [`PageTraceListDto`](#schema-pagetracelistdto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`PageTraceListDto`](#schema-pagetracelistdto) |
| `401` | Unauthorized | [`PageTraceListDto`](#schema-pagetracelistdto) |
| `404` | Agent/Team not found | [`PageTraceListDto`](#schema-pagetracelistdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/traces/session-stats`

**Get trace session statistics**

Retrieves aggregated trace statistics grouped by session.

            **Required Filters**: Exactly one of the following method must be provided:
            - `agentId`: Statistics for this agent
            - `teamId`: Statistics for this team

            **Optional Filters**: userId, start_time, end_time

- **Auth:** `bearer-jwt`
- **operationId:** `getTraceSessionStats_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `userId` | query |  | string |  |
| `startTime` | query |  | string |  |
| `endTime` | query |  | string |  |
| `page` | query |  | integer (int32) |  |
| `limit` | query |  | integer (int32) |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Statistics retrieved successfully | [`PageTraceSessionStatsDto`](#schema-pagetracesessionstatsdto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`PageTraceSessionStatsDto`](#schema-pagetracesessionstatsdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/traces/{traceId}`

**Get trace detail**

Retrieves detailed information about a specific trace.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Trace belongs to this agent
            - `teamId`: Trace belongs to this team

- **Auth:** `bearer-jwt`
- **operationId:** `getTrace_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `traceId` | path | yes | string |  |
| `agentId` | query |  | string |  |
| `teamId` | query |  | string |  |
| `spanId` | query |  | string |  |
| `runId` | query |  | string |  |
| `dbId` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Trace retrieved successfully | [`TraceDetailDto`](#schema-tracedetaildto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`TraceDetailDto`](#schema-tracedetaildto) |
| `404` | Trace not found | [`TraceDetailDto`](#schema-tracedetaildto) |


---

<a id="dom-governance-policies-compliance"></a>

### Governance: Policies & Compliance

_16 operations._

#### ▸ Compliance Frameworks

_Compliance framework management endpoints_

#### `GET /api/compliance-frameworks`

**List compliance frameworks**

Returns all preloaded compliance frameworks available in the system

- **Auth:** `bearer-jwt`
- **operationId:** `listComplianceFrameworks`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successfully retrieved compliance frameworks | [`ComplianceFrameworkDTO`](#schema-complianceframeworkdto)[] |
| `401` | Unauthorized - authentication required | [`ComplianceFrameworkDTO`](#schema-complianceframeworkdto)[] |


#### ▸ Policy Statistics

_API for policy enforcement statistics_

#### `GET /api/organizations/{orgId}/policies/statistics`

**Get policy statistics**

Returns policy trigger statistics by scope, wrapped with summary.

            Cumulative statistics (fast): Returns total trigger counts from the counter table.
            Time-range statistics: Returns trigger counts within the specified date range from audit logs.

            Statistics are filtered to effective policies only. If frameworkId is provided,
            further filters to policies belonging to that framework.

            Requires audit.read permission.

- **Auth:** **Public** — no authentication required
- **operationId:** `getStatistics_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `orgId` | path | yes | integer (int64) |  |
| `projectId` | query |  | integer (int64) | Filter by project ID |
| `agentId` | query |  | integer (int64) | Filter by agent ID |
| `teamId` | query |  | integer (int64) | Filter by team ID |
| `frameworkId` | query |  | integer (int64) | Filter by compliance framework ID |
| `fromDate` | query |  | string (date-time) | Start of date range for time-range queries (ISO-8601 format) |
| `toDate` | query |  | string (date-time) | End of date range for time-range queries (ISO-8601 format) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PolicyStatisticsResponse`](#schema-policystatisticsresponse) |


#### ▸ Policies

_Policy management endpoints_

#### `GET /api/organizations/{organizationId}/policies`

**List policies with context-specific enabled status**

Returns paginated list of policies (default + organization-specific) with their isEnabled flag
            set based on the specified context. Applies hierarchical precedence: Agent > Project > Organization.

            Parameters:
            - projectId: Optional project ID for project/agent-level context
            - agentId: Optional agent ID for agent-level context (requires projectId)
            - isDefault: Filter by default status (true/false)
            - frameworkId: Optional filter by compliance framework ID
            - search: Optional search term for policy name (case-insensitive)
            - sortBy: Sort field (name, created, updated), default: name
            - direction: Sort direction (asc, desc), default: asc

            Context examples:
            - Organization level: No projectId/agentId
            - Project level: ?projectId=123
            - Agent level: ?projectId=123&agentId=456

- **Auth:** `bearer-jwt`
- **operationId:** `listPoliciesWithContextStatus`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | query |  | integer (int64) | Project ID for project/team/agent-level context (optional, auto-derived from teamId/agentId if not provided) |
| `teamId` | query |  | integer (int64) | Team ID for team-level context (optional, projectId auto-derived if not provided) |
| `agentId` | query |  | integer (int64) | Agent ID for agent-level context (optional, projectId auto-derived if not provided) |
| `isDefault` | query |  | boolean | Filter by default status (true/false, optional) |
| `frameworkId` | query |  | integer (int64) | Filter by compliance framework ID (optional) |
| `search` | query |  | string | Search policies by name (case-insensitive partial match, optional) |
| `sortBy` | query |  | string | Sort by field. Allowed fields: name, created, updated |
| `direction` | query |  | string | Sort direction (asc or desc) |
| `page` | query |  | integer | Zero-based page index |
| `size` | query |  | integer | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successfully retrieved policies with context-specific enabled status | [`Page`](#schema-page) |
| `400` | Bad Request - invalid pagination parameters or missing projectId for agent context | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |
| `401` | Unauthorized - authentication required | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |
| `403` | Forbidden - insufficient permissions | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |


#### `POST /api/organizations/{organizationId}/policies`

**Create organization policy**

Create a new organization-specific policy (org admin only)

- **Auth:** `bearer-jwt`
- **operationId:** `createOrgPolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |

**Request body** (`application/json`)

Schema: [`PolicyModel`](#schema-policymodel)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `frameworks` | integer (int64)[] | yes |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Policy created successfully | [`PolicyItemDTO`](#schema-policyitemdto) |
| `400` | Bad Request - invalid request body or validation error | [`PolicyItemDTO`](#schema-policyitemdto) |
| `401` | Unauthorized - authentication required | [`PolicyItemDTO`](#schema-policyitemdto) |
| `403` | Forbidden - insufficient permissions | [`PolicyItemDTO`](#schema-policyitemdto) |
| `404` | Not Found - organization not found | [`PolicyItemDTO`](#schema-policyitemdto) |
| `409` | Conflict - policy with this name already exists in this organization | [`PolicyItemDTO`](#schema-policyitemdto) |


#### `GET /api/organizations/{organizationId}/policies/status`

**Get effective enabled policies**

Returns paginated list of policies that are effectively enabled for the given context.
            Applies hierarchical precedence: Agent > Project > Organization.

            Contexts:
            - Organization level: No query params
            - Project level: ?projectId=123
            - Agent level: ?agentId=456&projectId=123 (both required for agent context)

- **Auth:** `bearer-jwt`
- **operationId:** `getEffectivePolicies`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `projectId` | query |  | integer (int64) | Project ID for project/team/agent-level context (optional, required if teamId or agentId is provided) |
| `teamId` | query |  | integer (int64) | Team ID for team-level context (optional, requires projectId) |
| `agentId` | query |  | integer (int64) | Agent ID for agent-level context (optional, requires projectId and teamId) |
| `page` | query |  | integer | Zero-based page index |
| `size` | query |  | integer | Page size (max 100) |
| `sortBy` | query |  | string | Sort by field. Allowed fields: name, created, updated |
| `direction` | query |  | string | Sort direction (asc or desc) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Successfully retrieved effective policies | [`Page`](#schema-page) |
| `400` | Bad Request - invalid pagination parameters or missing projectId for agent context | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |
| `401` | Unauthorized - authentication required | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |
| `403` | Forbidden - insufficient permissions | [`PagePolicyItemDTO`](#schema-pagepolicyitemdto) |


#### `PUT /api/organizations/{organizationId}/policies/{policyId}`

**Update organization policy**

Update an existing organization-specific policy (org admin only)

- **Auth:** `bearer-jwt`
- **operationId:** `updateOrgPolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `policyId` | path | yes | integer (int64) | Policy ID |

**Request body** (`application/json`)

Schema: [`PolicyModel`](#schema-policymodel)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `frameworks` | integer (int64)[] | yes |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Policy updated successfully | [`PolicyItemDTO`](#schema-policyitemdto) |
| `400` | Bad Request - invalid request body or validation error | [`PolicyItemDTO`](#schema-policyitemdto) |
| `401` | Unauthorized - authentication required | [`PolicyItemDTO`](#schema-policyitemdto) |
| `403` | Forbidden - insufficient permissions | [`PolicyItemDTO`](#schema-policyitemdto) |
| `404` | Not Found - policy not found or doesn't belong to organization | [`PolicyItemDTO`](#schema-policyitemdto) |
| `409` | Conflict - policy with this name already exists in this organization | [`PolicyItemDTO`](#schema-policyitemdto) |


#### `DELETE /api/organizations/{organizationId}/policies/{policyId}`

**Delete organization policy**

Delete an existing organization-specific policy (org admin only). Removes all framework associations and enforcement records.

- **Auth:** `bearer-jwt`
- **operationId:** `deleteOrgPolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `policyId` | path | yes | integer (int64) | Policy ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Policy deleted successfully |  |
| `400` | Bad Request - policy is not an organization policy |  |
| `401` | Unauthorized - authentication required |  |
| `403` | Forbidden - insufficient permissions |  |
| `404` | Not Found - policy not found or doesn't belong to organization |  |


#### `PUT /api/organizations/{organizationId}/policies/{policyId}/status`

**Enable policy at organization, project, or agent level**

Enable a policy at different hierarchical scopes:
            - Organization level: No query params
            - Project level: ?projectId=123
            - Agent level: ?agentId=456 (projectId is ignored for agent level)

            Idempotent: Enabling already-enabled policy returns success.

            Default policies can be enabled by any organization.
            Organization-specific policies can only be enabled within their organization.

- **Auth:** `bearer-jwt`
- **operationId:** `enablePolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `policyId` | path | yes | integer (int64) | Policy ID |
| `projectId` | query |  | integer (int64) | Project ID for project-level enforcement (optional) |
| `teamId` | query |  | integer (int64) | Team ID for team-level enforcement (optional). When specified, projectId is ignored. |
| `agentId` | query |  | integer (int64) | Agent ID for agent-level enforcement (optional). When specified, projectId and teamId are ignored. |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Policy enabled successfully | [`PolicyEnforcementDTO`](#schema-policyenforcementdto) |
| `400` | Bad Request - invalid projectId or agentId | [`PolicyEnforcementDTO`](#schema-policyenforcementdto) |
| `401` | Unauthorized - authentication required | [`PolicyEnforcementDTO`](#schema-policyenforcementdto) |
| `403` | Forbidden - requires org admin permission | [`PolicyEnforcementDTO`](#schema-policyenforcementdto) |
| `404` | Not Found - policy not found or not accessible by organization | [`PolicyEnforcementDTO`](#schema-policyenforcementdto) |


#### `DELETE /api/organizations/{organizationId}/policies/{policyId}/status`

**Disable policy at organization, project, or agent level**

Disable a policy at different hierarchical scopes:
            - Organization level: No query params
            - Project level: ?projectId=123
            - Agent level: ?agentId=456 (projectId is ignored for agent level)

            Idempotent: Disabling already-disabled policy returns success.

- **Auth:** `bearer-jwt`
- **operationId:** `disablePolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |
| `policyId` | path | yes | integer (int64) | Policy ID |
| `projectId` | query |  | integer (int64) | Project ID for project-level enforcement (optional) |
| `teamId` | query |  | integer (int64) | Team ID for team-level enforcement (optional). When specified, projectId is ignored. |
| `agentId` | query |  | integer (int64) | Agent ID for agent-level enforcement (optional). When specified, projectId and teamId are ignored. |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Policy disabled successfully (no content) |  |
| `400` | Bad Request - invalid projectId or agentId |  |
| `401` | Unauthorized - authentication required |  |
| `403` | Forbidden - requires org admin permission |  |
| `404` | Not Found - policy not found or not accessible by organization |  |


#### `POST /api/policies`

**Create default policy**

Create a new default policy available to all organizations (super admin only)

- **Auth:** `bearer-jwt`
- **operationId:** `createDefaultPolicy`

**Request body** (`application/json`)

Schema: [`PolicyModel`](#schema-policymodel)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `frameworks` | integer (int64)[] | yes |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Policy created successfully | [`PolicyItemDTO`](#schema-policyitemdto) |
| `400` | Bad Request - invalid request body or validation error | [`PolicyItemDTO`](#schema-policyitemdto) |
| `401` | Unauthorized - authentication required | [`PolicyItemDTO`](#schema-policyitemdto) |
| `403` | Forbidden - requires super admin role | [`PolicyItemDTO`](#schema-policyitemdto) |
| `409` | Conflict - policy with this name already exists | [`PolicyItemDTO`](#schema-policyitemdto) |


#### `PUT /api/policies/{id}`

**Update default policy**

Update an existing default policy (super admin only)

- **Auth:** `bearer-jwt`
- **operationId:** `updateDefaultPolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`PolicyModel`](#schema-policymodel)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `frameworks` | integer (int64)[] | yes |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Policy updated successfully | [`PolicyItemDTO`](#schema-policyitemdto) |
| `400` | Bad Request - invalid request body or validation error | [`PolicyItemDTO`](#schema-policyitemdto) |
| `401` | Unauthorized - authentication required | [`PolicyItemDTO`](#schema-policyitemdto) |
| `403` | Forbidden - requires super admin role | [`PolicyItemDTO`](#schema-policyitemdto) |
| `404` | Not Found - policy not found | [`PolicyItemDTO`](#schema-policyitemdto) |
| `409` | Conflict - policy with this name already exists | [`PolicyItemDTO`](#schema-policyitemdto) |


#### `DELETE /api/policies/{id}`

**Delete default policy**

Delete an existing default policy (super admin only). Removes all framework associations and enforcement records.

- **Auth:** `bearer-jwt`
- **operationId:** `deleteDefaultPolicy`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Policy ID to delete |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Policy deleted successfully |  |
| `400` | Bad Request - policy is not a default policy |  |
| `401` | Unauthorized - authentication required |  |
| `403` | Forbidden - requires super admin role |  |
| `404` | Not Found - policy not found |  |


#### ▸ Policy Test

_Test policy configurations against sample text_

#### `POST /api/policy-test`

**Test policy configuration**

Test how a policy configuration would process sample text without saving the policy.
            Supports all policy actions: REDACT, BLOCK, TRANSFORM, and ALERT.

            Example request:
            {
              "text": "My SSN is 123-45-6789",
              "stage": "PRE_PROMPT",
              "type": "REGEX",
              "regex": "\\d{3}-\\d{2}-\\d{4}",
              "action": "REDACT",
              "redactConfig": {
                "type": "MASK",
                "mask": "*"
              }
            }

- **Auth:** `bearer-jwt`
- **operationId:** `testPolicy`

**Request body** (`application/json`)

Schema: [`PolicyTestRequest`](#schema-policytestrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `text` | string | yes | minLength=`1` |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Test completed successfully | [`PolicyTestResponse`](#schema-policytestresponse) |
| `400` | Bad Request - invalid request body or validation error | [`PolicyTestResponse`](#schema-policytestresponse) |
| `401` | Unauthorized - authentication required | [`PolicyTestResponse`](#schema-policytestresponse) |


#### ▸ Policy Audit Logs

_API for managing policy enforcement audit logs_

#### `GET /api/v1/organizations/{orgId}/policy-audit-logs`

**List policy audit logs**

Retrieves policy audit logs with filtering and pagination. Requires audit.read permission.

- **Auth:** **Public** — no authentication required
- **operationId:** `listAuditLogs`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `orgId` | path | yes | integer (int64) |  |
| `fromDate` | query |  | string (date-time) | Start of date range (ISO-8601 format) |
| `toDate` | query |  | string (date-time) | End of date range (ISO-8601 format) |
| `policyId` | query |  | integer (int64) | Filter by numeric policy ID (service will lookup reference ID) |
| `policyReferenceId` | query |  | string | Filter by policy reference ID (direct query) |
| `action` | query |  | string | Filter by action type |
| `stage` | query |  | string | Filter by evaluation stage |
| `projectId` | query |  | integer (int64) | Filter by project ID |
| `agentId` | query |  | integer (int64) | Filter by agent ID |
| `callerType` | query |  | string | Filter by caller type |
| `frameworkId` | query |  | integer (int64) | Filter by compliance framework ID |
| `page` | query |  | integer (int32) | Page number (0-indexed) |
| `size` | query |  | integer (int32) | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PagePolicyAuditLogDTO`](#schema-pagepolicyauditlogdto) |


#### `GET /api/v1/organizations/{orgId}/policy-audit-logs/export`

**Export policy audit logs**

Exports policy audit logs in CSV or JSON format. Requires reports.generate permission.

- **Auth:** **Public** — no authentication required
- **operationId:** `exportAuditLogs`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `orgId` | path | yes | integer (int64) |  |
| `format` | query | yes | string | Export format (csv or json) |
| `fromDate` | query |  | string (date-time) | Start of date range (ISO-8601 format) |
| `toDate` | query |  | string (date-time) | End of date range (ISO-8601 format) |
| `policyId` | query |  | integer (int64) | Filter by numeric policy ID |
| `policyReferenceId` | query |  | string | Filter by policy reference ID |
| `action` | query |  | string | Filter by action type |
| `stage` | query |  | string | Filter by evaluation stage |
| `projectId` | query |  | integer (int64) | Filter by project ID |
| `agentId` | query |  | integer (int64) | Filter by agent ID |
| `callerType` | query |  | string | Filter by caller type |
| `frameworkId` | query |  | integer (int64) | Filter by compliance framework ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `GET /api/v1/organizations/{orgId}/policy-audit-logs/statistics`

**Get audit log statistics**

Returns statistics about policy audit logs for an organization. Requires audit.read permission.

- **Auth:** **Public** — no authentication required
- **operationId:** `getStatistics`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `orgId` | path | yes | integer (int64) |  |
| `fromDate` | query |  | string (date-time) | Start of date range (ISO-8601 format) |
| `toDate` | query |  | string (date-time) | End of date range (ISO-8601 format) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PolicyAuditLogStatistics`](#schema-policyauditlogstatistics) |


---

<a id="dom-webhooks-eventing"></a>

### Webhooks & Eventing

_11 operations._

#### ▸ Stripe webhook

_Expose Stripe webhook endpoints_

#### `POST /api/webhook/stripe`

**handleStripeWebhook**

- **Auth:** **Public** — no authentication required
- **operationId:** `handleStripeWebhook`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `Stripe-Signature` | header | yes | string |  |

**Request body** (`application/json`)

```json
{
  "type": "string"
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### ▸ Webhooks v1

_Webhook subscription management endpoints following Standard Webhooks specification_

#### `GET /api/webhooks/deliveries`

**List webhook deliveries**

Get webhook delivery history with filtering options for troubleshooting and monitoring

- **Auth:** `bearer-token`
- **operationId:** `listWebhookDeliveries`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organization_id` | query | yes | integer (int64) | Organization ID |
| `project_id` | query |  | integer (int64) | Filter by project ID (optional) |
| `subscription_id` | query |  | integer (int64) | Filter by subscription ID (optional) |
| `session_id` | query |  | string | Filter by session ID (optional) |
| `event_type` | query |  | string | Filter by event type (optional) |
| `status` | query |  | string | Filter by delivery status (optional) |
| `period_start` | query |  | string | Filter deliveries created after this date (ISO 8601) |
| `period_end` | query |  | string | Filter deliveries created before this date (ISO 8601) |
| `page` | query |  | integer (int32) | Page number (0-based) |
| `size` | query |  | integer (int32) | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook deliveries retrieved successfully | [`PageWebhookDeliveryResponse`](#schema-pagewebhookdeliveryresponse) |
| `401` | Invalid JWT token | [`PageWebhookDeliveryResponse`](#schema-pagewebhookdeliveryresponse) |
| `403` | Insufficient permissions | [`PageWebhookDeliveryResponse`](#schema-pagewebhookdeliveryresponse) |
| `429` | Rate limit exceeded | [`PageWebhookDeliveryResponse`](#schema-pagewebhookdeliveryresponse) |


#### `POST /api/webhooks/deliveries/{id}/replay`

**Replay webhook delivery**

Manually replay a webhook delivery for troubleshooting or recovery purposes

- **Auth:** `bearer-token`
- **operationId:** `replayWebhookDelivery`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Webhook delivery ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |

**Request body** (`application/json`)

Schema: [`WebhookReplayRequest`](#schema-webhookreplayrequest)

> Webhook replay request

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `force` | boolean | yes |  | Force replay even if delivery was successful |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook delivery replay initiated successfully | `string` |
| `400` | Invalid request parameters | `string` |
| `401` | Invalid JWT token | `string` |
| `403` | Insufficient permissions | `string` |
| `404` | Webhook delivery not found | `string` |
| `429` | Rate limit exceeded | `string` |


#### `GET /api/webhooks/event-types`

**List supported event types**

Get all supported webhook event types that can be subscribed to

- **Auth:** `bearer-token`
- **operationId:** `listEventTypes`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Event types retrieved successfully | string[] |
| `401` | Invalid JWT token | string[] |
| `429` | Rate limit exceeded | string[] |


#### `GET /api/webhooks/subscriptions`

**List webhook subscriptions**

List all webhook subscriptions for the specified organization and optionally filter by project

- **Auth:** `bearer-token`
- **operationId:** `listWebhookSubscriptions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organization_id` | query | yes | integer (int64) | Organization ID |
| `project_id` | query |  | integer (int64) | Filter by project ID (optional) |
| `page` | query |  | integer (int32) | Page number (0-based) |
| `size` | query |  | integer (int32) | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook subscriptions retrieved successfully | [`PageWebhookSubscriptionResponse`](#schema-pagewebhooksubscriptionresponse) |
| `401` | Invalid JWT token | [`PageWebhookSubscriptionResponse`](#schema-pagewebhooksubscriptionresponse) |
| `403` | Insufficient permissions | [`PageWebhookSubscriptionResponse`](#schema-pagewebhooksubscriptionresponse) |
| `429` | Rate limit exceeded | [`PageWebhookSubscriptionResponse`](#schema-pagewebhooksubscriptionresponse) |


#### `POST /api/webhooks/subscriptions`

**Create webhook subscription**

Create a new webhook subscription for receiving Standard Webhooks notifications

- **Auth:** `bearer-token`
- **operationId:** `createWebhookSubscription`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organization_id` | query | yes | integer (int64) | Organization ID |
| `project_id` | query | yes | integer (int64) | Project ID |

**Request body** (`application/json`)

Schema: [`CreateWebhookSubscriptionRequest`](#schema-createwebhooksubscriptionrequest)

> Request to create a webhook subscription

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`255` | Name of the webhook subscription |
| `endpointUrl` | string | yes | minLength=`0` ; maxLength=`2048` ; pattern=`^https?://.*` | The webhook endpoint URL |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Description of the webhook subscription |
| `eventTypes` | string[] | yes |  | List of event types to subscribe to |
| `maxDeliveryAttempts` | integer (int32) | yes | minimum=`1` ; maximum=`10` | Maximum number of delivery attempts |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Webhook subscription created successfully | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `400` | Invalid request parameters | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `401` | Invalid JWT token | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `403` | Insufficient permissions | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `409` | Webhook subscription with this name already exists | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `429` | Rate limit exceeded | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |


#### `GET /api/webhooks/subscriptions/{id}`

**Get webhook subscription**

Retrieve details of a specific webhook subscription by ID

- **Auth:** `bearer-token`
- **operationId:** `getWebhookSubscription`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Webhook subscription ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook subscription retrieved successfully | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `401` | Invalid JWT token | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `403` | Insufficient permissions | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `404` | Webhook subscription not found | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `429` | Rate limit exceeded | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |


#### `PUT /api/webhooks/subscriptions/{id}`

**Update webhook subscription**

Update an existing webhook subscription configuration

- **Auth:** `bearer-token`
- **operationId:** `updateWebhookSubscription`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Webhook subscription ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |

**Request body** (`application/json`)

Schema: [`UpdateWebhookSubscriptionRequest`](#schema-updatewebhooksubscriptionrequest)

> Request to update a webhook subscription

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`255` | Name of the webhook subscription |
| `endpointUrl` | string |  | minLength=`0` ; maxLength=`2048` ; pattern=`^https?://.*` | The webhook endpoint URL |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Description of the webhook subscription |
| `eventTypes` | string[] |  |  | List of event types to subscribe to |
| `maxDeliveryAttempts` | integer (int32) |  | minimum=`1` ; maximum=`10` | Maximum number of delivery attempts |
| `isActive` | boolean |  |  | Whether the subscription is active |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook subscription updated successfully | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `400` | Invalid request parameters | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `401` | Invalid JWT token | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `403` | Insufficient permissions | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `404` | Webhook subscription not found | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `409` | Webhook subscription with this name already exists | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |
| `429` | Rate limit exceeded | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse) |


#### `DELETE /api/webhooks/subscriptions/{id}`

**Delete webhook subscription**

Delete a webhook subscription and stop receiving notifications

- **Auth:** `bearer-token`
- **operationId:** `deleteWebhookSubscription`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Webhook subscription ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Webhook subscription deleted successfully |  |
| `401` | Invalid JWT token |  |
| `403` | Insufficient permissions |  |
| `404` | Webhook subscription not found |  |
| `429` | Rate limit exceeded |  |


#### `POST /api/webhooks/subscriptions/{id}/test`

**Test webhook endpoint**

Send a test payload to a webhook subscription endpoint to verify connectivity and configuration

- **Auth:** `bearer-token`
- **operationId:** `testWebhookSubscription`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Webhook subscription ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |
| `event_type` | query | yes | string | Event Type |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook test completed (check success field for actual result) | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `400` | Invalid request parameters | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `401` | Invalid JWT token | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `403` | Insufficient permissions | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `404` | Webhook subscription not found | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `409` | Webhook subscription is inactive | [`WebhookTestResponse`](#schema-webhooktestresponse) |
| `429` | Rate limit exceeded | [`WebhookTestResponse`](#schema-webhooktestresponse) |


#### `GET /api/webhooks/subscriptions/{subscriptionId}/statistics`

**Get webhook delivery statistics**

Get delivery statistics and metrics for monitoring webhook health with optional date range filtering

- **Auth:** `bearer-token`
- **operationId:** `getWebhookDeliveryStatistics`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `subscriptionId` | path | yes | integer (int64) | Webhook subscription ID |
| `organization_id` | query | yes | integer (int64) | Organization ID |
| `period_start` | query |  | string (date-time) | Filter statistics for deliveries created after this date (ISO 8601 format) |
| `period_end` | query |  | string (date-time) | Filter statistics for deliveries created before this date (ISO 8601 format) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Webhook delivery statistics retrieved successfully | [`WebhookDeliveryStats`](#schema-webhookdeliverystats) |
| `401` | Invalid JWT token | [`WebhookDeliveryStats`](#schema-webhookdeliverystats) |
| `403` | Insufficient permissions | [`WebhookDeliveryStats`](#schema-webhookdeliverystats) |
| `404` | Subscription not found | [`WebhookDeliveryStats`](#schema-webhookdeliverystats) |
| `429` | Rate limit exceeded | [`WebhookDeliveryStats`](#schema-webhookdeliverystats) |


---

<a id="dom-billing"></a>

### Billing

_14 operations._

#### ▸ Billing

_Manage billing endpoints_

#### `GET /api/organizations/{organizationId}/billing`

**getBillingInfo**

- **Auth:** `bearer-jwt`
- **operationId:** `getBillingInfo`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`CustomerBillingDto`](#schema-customerbillingdto) |


#### `PUT /api/organizations/{organizationId}/billing/auto-recharge`

**updateAutoRechargeSettings**

- **Auth:** `bearer-jwt`
- **operationId:** `updateAutoRechargeSettings`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`AutoRechargeConfigRequest`](#schema-autorechargeconfigrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `enabled` | boolean | yes |  |  |
| `threshold` | integer (int64) | yes |  |  |
| `amount` | integer (int64) | yes |  |  |
| `monthlyLimit` | integer (int64) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AutoRechargeConfigResponse`](#schema-autorechargeconfigresponse) |


#### `POST /api/organizations/{organizationId}/billing/balance/add`

**addFunds**

- **Auth:** `bearer-jwt`
- **operationId:** `addFunds`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`AddFundsRequest`](#schema-addfundsrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `amount` | integer (int64) | yes |  |  |
| `paymentMethodId` | integer (int64) | yes |  |  |
| `idempotencyKey` | string | yes |  |  |
| `description` | string |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AddFundsResponse`](#schema-addfundsresponse) |


#### `GET /api/organizations/{organizationId}/billing/history`

**getPaymentTransactions**

- **Auth:** `bearer-jwt`
- **operationId:** `getPaymentTransactions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `page` | query |  | integer (int32) |  |
| `size` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `direction` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PagePaymentTransactionDto`](#schema-pagepaymenttransactiondto) |


#### `GET /api/organizations/{organizationId}/billing/payment-methods`

**getPaymentMethods**

- **Auth:** `bearer-jwt`
- **operationId:** `getPaymentMethods`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PaymentMethodDto`](#schema-paymentmethoddto)[] |


#### `POST /api/organizations/{organizationId}/billing/payment-methods`

**addPaymentMethod**

- **Auth:** `bearer-jwt`
- **operationId:** `addPaymentMethod`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`AddPaymentMethodRequest`](#schema-addpaymentmethodrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `paymentMethodId` | string | yes |  |  |
| `setAsDefault` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `DELETE /api/organizations/{organizationId}/billing/payment-methods/{paymentMethodId}`

**deletePaymentMethod**

- **Auth:** `bearer-jwt`
- **operationId:** `deletePaymentMethod`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `paymentMethodId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `PUT /api/organizations/{organizationId}/billing/payment-methods/{paymentMethodId}/default`

**setDefaultPaymentMethod**

- **Auth:** `bearer-jwt`
- **operationId:** `setDefaultPaymentMethod`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `paymentMethodId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### `GET /api/organizations/{organizationId}/billing/preferences`

**getBillingPreferences**

- **Auth:** `bearer-jwt`
- **operationId:** `getBillingPreferences`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`CustomerBillingInfo`](#schema-customerbillinginfo) |


#### `PUT /api/organizations/{organizationId}/billing/preferences`

**updateBillingPreferences**

- **Auth:** `bearer-jwt`
- **operationId:** `updateBillingPreferences`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`CustomerBillingInfo`](#schema-customerbillinginfo)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `companyName` | string |  |  |  |
| `billingEmail` | string |  |  |  |
| `country` | string |  |  |  |
| `address1` | string |  |  |  |
| `address2` | string |  |  |  |
| `city` | string |  |  |  |
| `state` | string |  |  |  |
| `zip` | string |  |  |  |
| `vat` | string |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`CustomerBillingInfo`](#schema-customerbillinginfo) |


#### `GET /api/organizations/{organizationId}/billing/setup-intent`

**getSetupIntentData**

- **Auth:** `bearer-jwt`
- **operationId:** `getSetupIntentData`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`SetupIntentResponse`](#schema-setupintentresponse) |


#### `GET /api/organizations/{organizationId}/billing/usage`

**getApiUsage_1**

- **Auth:** `bearer-jwt`
- **operationId:** `getApiUsage_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `scope` | query |  | string |  |
| `usageMode` | query |  | string |  |
| `projectIds` | query |  | integer (int64)[] |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |
| `userIds` | query |  | integer (int64)[] |  |
| `apiKeyIds` | query |  | integer (int64)[] |  |
| `granularity` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`UsageResponseDto`](#schema-usageresponsedto) |


#### `GET /api/organizations/{organizationId}/billing/usage/export`

**exportUsage_1**

- **Auth:** `bearer-jwt`
- **operationId:** `exportUsage_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `scope` | query |  | string |  |
| `usageMode` | query |  | string |  |
| `projectIds` | query |  | integer (int64)[] |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |
| `userIds` | query |  | integer (int64)[] |  |
| `apiKeyIds` | query |  | integer (int64)[] |  |
| `granularity` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `GET /api/organizations/{organizationId}/billing/usage/users`

**getUsageFilterUsers**

- **Auth:** `bearer-jwt`
- **operationId:** `getUsageFilterUsers`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `scope` | query |  | string |  |
| `projectIds` | query |  | integer (int64)[] |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`BaseUserDto`](#schema-baseuserdto)[] |


---

<a id="dom-platform-config-media"></a>

### Platform, Config & Media

_97 operations._

#### ▸ System Configuration

_System-wide configuration endpoints_

#### `GET /api/config/storage-upload`

**Get storage upload configuration**

Returns file upload limits and supported file types from the storage service

- **Auth:** `bearer-jwt`
- **operationId:** `getStorageUploadConfig`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Configuration retrieved successfully | [`StorageUploadConfigResponse`](#schema-storageuploadconfigresponse) |
| `503` | Storage service unavailable | [`StorageUploadConfigResponse`](#schema-storageuploadconfigresponse) |


#### ▸ Documentation Management

_Internal API endpoints for managing documentation content_

#### `GET /api/documentation/sections`

**Get all documentation sections**

- **Auth:** `bearer-token`
- **operationId:** `getAllSections`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | query |  | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`DocumentationSectionDto`](#schema-documentationsectiondto)[] |


#### `GET /api/documentation/sections/{sectionSlug}`

**Get a specific documentation section with subsections**

- **Auth:** `bearer-token`
- **operationId:** `getSection`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `sectionSlug` | path | yes | string |  |
| `organizationId` | query |  | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`DocumentationSectionWithFullSubsectionsDto`](#schema-documentationsectionwithfullsubsectionsdto) |


#### `GET /api/documentation/sections/{sectionSlug}/{subsectionSlug}`

**Get a specific documentation subsection**

- **Auth:** `bearer-token`
- **operationId:** `getSubsection`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `sectionSlug` | path | yes | string |  |
| `subsectionSlug` | path | yes | string |  |
| `organizationId` | query |  | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`DocumentationSubsectionDto`](#schema-documentationsubsectiondto) |


#### `PUT /api/documentation/sections/{sectionSlug}/{subsectionSlug}`

**Update a documentation subsection**

- **Auth:** `bearer-token`
- **operationId:** `updateSubsection`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `sectionSlug` | path | yes | string |  |
| `subsectionSlug` | path | yes | string |  |
| `organizationId` | query |  | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`UpdateDocumentationRequest`](#schema-updatedocumentationrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `content` | string | yes |  |  |
| `contentType` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`DocumentationUpdateResponse`](#schema-documentationupdateresponse) |


#### ▸ Image Proxy

_Proxy for serving images with permanent URLs_

#### `GET /api/images/{documentId}/{filename}`

**Get image**

Proxies image request to Storage Service. Storage Service looks up document in DB to get vector_store_id, then streams image from S3. This endpoint never expires, unlike direct S3 presigned URLs.

- **Auth:** **Public** — no authentication required
- **operationId:** `getImage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `documentId` | path | yes | string |  |
| `filename` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### ▸ Media Proxy

_Authenticated proxy for AI-uploaded media files_

#### `GET /api/media/{s3KeyEncoded}`

**Get media file**

Decodes the base64url S3 key and streams the object from the AI media bucket with its stored Content-Type. Responses are cacheable for one year (immutable).

- **Auth:** **Public** — no authentication required
- **operationId:** `getMedia`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `s3KeyEncoded` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### ▸ Public Configuration

_Public endpoints that do not require authentication_

#### `GET /api/public/config`

**Get application configuration**

Returns public application configuration including deployment mode and base URL

- **Auth:** **Public** — no authentication required
- **operationId:** `getConfig`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Configuration retrieved successfully | [`AppConfigDto`](#schema-appconfigdto) |


#### `GET /api/public/sip-url`

**Get SIP URL**

Returns the SIP URL for telephony preset configuration

- **Auth:** **Public** — no authentication required
- **operationId:** `getSipUrl`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | SIP URL retrieved successfully | [`SipUrlResponseDto`](#schema-sipurlresponsedto) |


#### ▸ API v1

_Public API v1 endpoints_

#### `GET /client/api/v1/agents`

**List available agents**

Returns a list of all available AI agents with their metadata, capabilities, and parameters

- **Auth:** `api-key`
- **operationId:** `listAgents`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agents retrieved successfully | [`ApiResponseListAgentResponse`](#schema-apiresponselistagentresponse) |
| `401` | Invalid API key | [`ApiResponseListAgentResponse`](#schema-apiresponselistagentresponse) |
| `429` | Rate limit exceeded | [`ApiResponseListAgentResponse`](#schema-apiresponselistagentresponse) |
| `503` | AI system unavailable | [`ApiResponseListAgentResponse`](#schema-apiresponselistagentresponse) |


#### `POST /client/api/v1/agents`

**Create a new agent**

Creates a new AI agent with the specified configuration

- **Auth:** `api-key`
- **operationId:** `createAgent`

**Request body** (`application/json`)

Schema: [`AgentV1CreateRequest`](#schema-agentv1createrequest)

> Request to create a new agent via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `projectRefId` | string |  |  | Reference ID of the project this agent belongs to (e.g. 'proj_...'). Use this instead of projectId. |
| `projectId` | integer (int64) |  |  | Numeric ID of the project. Deprecated — use projectRefId. |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Agent created successfully | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `400` | Invalid request parameters | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `401` | Invalid API key | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `429` | Rate limit exceeded | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `503` | AI system unavailable | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |


#### `GET /client/api/v1/agents/jobs/{jobId}/status`

**Get agent job status**

Returns the status, progress, and results of an asynchronous agent execution job. Use include_messages=true to get full conversation history.

- **Auth:** `api-key`
- **operationId:** `getJobStatus_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `jobId` | path | yes | string |  |
| `include_messages` | query |  | boolean | Include full message history in response (default: false for backward compatibility and performance) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Job status retrieved successfully | [`AgentJobResponseDto`](#schema-agentjobresponsedto) |
| `404` | Job not found | [`AgentJobResponseDto`](#schema-agentjobresponsedto) |
| `401` | Invalid API key | [`AgentJobResponseDto`](#schema-agentjobresponsedto) |
| `429` | Rate limit exceeded | [`AgentJobResponseDto`](#schema-agentjobresponsedto) |


#### `GET /client/api/v1/agents/llm-providers`

**Get configured LLM providers for agents**

Retrieves all LLM provider configurations for the organization with their available models. Models are filtered based on the enabled and custom models configured for each provider.

- **Auth:** `api-key`
- **operationId:** `getConfiguredLlmProviders_1`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |


#### `GET /client/api/v1/agents/options`

**agentOptions**

- **Auth:** `api-key`
- **operationId:** `agentOptions`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`AgentOptionV1Dto`](#schema-agentoptionv1dto)[] |


#### `GET /client/api/v1/agents/{agentId}`

**Get agent by ID**

Retrieves detailed information about a specific agent

- **Auth:** `api-key`
- **operationId:** `getAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent retrieved successfully | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `401` | Invalid API key | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `404` | Agent not found | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `429` | Rate limit exceeded | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |


#### `PUT /client/api/v1/agents/{agentId}`

**Update an agent**

Updates an existing agent's configuration

- **Auth:** `api-key`
- **operationId:** `updateAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |

**Request body** (`application/json`)

Schema: [`AgentV1UpdateRequest`](#schema-agentv1updaterequest)

> Request to update an existing agent via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent updated successfully | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `400` | Invalid request parameters | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `401` | Invalid API key | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `404` | Agent not found | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `429` | Rate limit exceeded | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `503` | AI system unavailable | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |


#### `DELETE /client/api/v1/agents/{agentId}`

**Delete an agent**

Deletes an existing agent permanently

- **Auth:** `api-key`
- **operationId:** `deleteAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Agent deleted successfully |  |
| `401` | Invalid API key |  |
| `404` | Agent not found |  |
| `429` | Rate limit exceeded |  |


#### `POST /client/api/v1/agents/{agentId}/clone`

**Clone an agent**

Creates a copy of an existing agent with a new name, inheriting all configuration from the source agent

- **Auth:** `api-key`
- **operationId:** `cloneAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string | Agent ID |

**Request body** (`application/json`)

Schema: [`AgentCloneRequest`](#schema-agentclonerequest)

> Request to clone an existing agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned agent |
| `presetName` | string |  | minLength=`1` ; maxLength=`100` | Display name for the cloned agent shown in the platform dashboard |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Agent cloned successfully | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `400` | Invalid request parameters | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `401` | Invalid API key | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `404` | Agent not found | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `429` | Rate limit exceeded | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |
| `503` | AI system unavailable | [`ApiResponseAgentResponse`](#schema-apiresponseagentresponse) |


#### `POST /client/api/v1/agents/{agentId}/run`

**Execute agent synchronously**

Executes the specified agent with given parameters and returns the result immediately

- **Auth:** `api-key`
- **operationId:** `executeAgent`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `tag` | query |  | string |  |
| `user_id` | query |  | string |  |
| `session_id` | query |  | string |  |
| `tool_context` | query |  | string |  |
| `include_messages` | query |  | boolean | Include full message history in response (default: false for backward compatibility) |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "audio": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "images": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "videos": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Agent executed successfully | [`ApiResponseAgentRunResponseData`](#schema-apiresponseagentrunresponsedata) |
| `400` | Invalid parameters or agent not found | [`ApiResponseAgentRunResponseData`](#schema-apiresponseagentrunresponsedata) |
| `401` | Invalid API key | [`ApiResponseAgentRunResponseData`](#schema-apiresponseagentrunresponsedata) |
| `429` | Rate limit exceeded | [`ApiResponseAgentRunResponseData`](#schema-apiresponseagentrunresponsedata) |
| `503` | AI system unavailable | [`ApiResponseAgentRunResponseData`](#schema-apiresponseagentrunresponsedata) |


#### `POST /client/api/v1/agents/{agentId}/run-async`

**Execute agent asynchronously**

Starts asynchronous execution of the specified agent and returns a job ID for status tracking

- **Auth:** `api-key`
- **operationId:** `executeAgentAsync`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `tag` | query |  | string |  |
| `user_id` | query |  | string |  |
| `session_id` | query |  | string |  |
| `tool_context` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "audio": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "images": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "videos": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `202` | Agent execution started | [`AgentJobCreatedDto`](#schema-agentjobcreateddto) |
| `400` | Invalid parameters or agent not found | [`AgentJobCreatedDto`](#schema-agentjobcreateddto) |
| `401` | Invalid API key | [`AgentJobCreatedDto`](#schema-agentjobcreateddto) |
| `429` | Rate limit exceeded | [`AgentJobCreatedDto`](#schema-agentjobcreateddto) |
| `503` | AI system unavailable | [`AgentJobCreatedDto`](#schema-agentjobcreateddto) |


#### `POST /client/api/v1/agents/{agentId}/run-stream`

**Execute agent with streaming response**

Executes the specified agent with streaming response for real-time output

- **Auth:** `api-key`
- **operationId:** `executeAgentStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string |  |
| `message` | query | yes | string |  |
| `tag` | query |  | string |  |
| `user_id` | query |  | string |  |
| `session_id` | query |  | string |  |
| `tool_context` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "audio": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "images": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "videos": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    },
    "files": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Streaming execution started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid parameters | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `401` | Invalid API key | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI system unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /client/api/v1/agents/{agentId}/runs/{runId}/cancel`

**Cancel an agent run**

Cancels an in-progress agent run. Usage is tracked after successful cancellation.

- **Auth:** `api-key`
- **operationId:** `cancelAgentRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string |  |
| `runId` | path | yes | string |  |
| `session_id` | query | yes | string |  |
| `user_id` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully | [`ApiResponseString`](#schema-apiresponsestring) |
| `401` | Invalid API key | [`ApiResponseString`](#schema-apiresponsestring) |
| `404` | Agent or run not found | [`ApiResponseString`](#schema-apiresponsestring) |
| `503` | AI system unavailable | [`ApiResponseString`](#schema-apiresponsestring) |


#### `POST /client/api/v1/agents/{agentId}/runs/{runId}/continue`

**Continue an agent run with streaming response**

Resumes a paused agent run (e.g. after tool-approval) and streams the response

- **Auth:** `api-key`
- **operationId:** `continueAgentRunStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | path | yes | string |  |
| `runId` | path | yes | string |  |
| `user_id` | query | yes | string |  |
| `session_id` | query | yes | string |  |
| `tools` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Continuation stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid parameters | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `401` | Invalid API key | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Agent or run not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI system unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `GET /client/api/v1/eval-runs`

**List evaluation runs**

- **Auth:** `api-key`
- **operationId:** `listEvalRuns`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `modelId` | query |  | string |  |
| `limit` | query |  | integer (int32) |  |
| `page` | query |  | integer (int32) |  |
| `sortBy` | query |  | string |  |
| `sortOrder` | query |  | string |  |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |
| `evalTypes` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval runs retrieved successfully | [`PagedResponseEvalRunListDto`](#schema-pagedresponseevalrunlistdto) |
| `400` | Missing or multiple component filters | [`PagedResponseEvalRunListDto`](#schema-pagedresponseevalrunlistdto) |
| `401` | Invalid or missing API key | [`PagedResponseEvalRunListDto`](#schema-pagedresponseevalrunlistdto) |
| `404` | Agent/Team not found | [`PagedResponseEvalRunListDto`](#schema-pagedresponseevalrunlistdto) |


#### `POST /client/api/v1/eval-runs`

**Execute evaluation**

- **Auth:** `api-key`
- **operationId:** `executeEval`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`EvalRunInputRequest`](#schema-evalruninputrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `eval_type` | string | yes |  |  |
| `input` | string | yes |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `model_id` | string |  |  |  |
| `model_provider` | string |  |  |  |
| `provider_ref_id` | string |  |  |  |
| `name` | string |  |  |  |
| `expected_output` | string |  |  |  |
| `criteria` | string |  |  |  |
| `scoring_strategy` | string |  |  |  |
| `threshold` | integer (int32) |  |  |  |
| `num_iterations` | integer (int32) | yes |  |  |
| `warmup_runs` | integer (int32) | yes |  |  |
| `additional_guidelines` | string |  |  |  |
| `additional_context` | string |  |  |  |
| `expected_tool_calls` | string[] |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Evaluation executed successfully | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `400` | Missing or multiple component filters | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `404` | Agent/Team not found | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |


#### `GET /client/api/v1/eval-runs/{evalRunId}`

**Get evaluation run**

- **Auth:** `api-key`
- **operationId:** `getEvalRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval run retrieved successfully | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `400` | Missing or multiple component filters | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `404` | Eval run not found | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |


#### `PATCH /client/api/v1/eval-runs/{evalRunId}`

**Update evaluation run**

- **Auth:** `api-key`
- **operationId:** `updateEvalRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`UpdateEvalRunRequest`](#schema-updateevalrunrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  |  |  |
| `eval_data` | object |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Eval run updated successfully | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `400` | Missing or multiple component filters | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |
| `404` | Eval run not found | [`ApiResponseEvalRunDetailDto`](#schema-apiresponseevalrundetaildto) |


#### `DELETE /client/api/v1/eval-runs/{evalRunId}`

**Delete evaluation run**

- **Auth:** `api-key`
- **operationId:** `deleteEvalRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `evalRunId` | path | yes | string |  |
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `dbId` | query |  | string |  |
| `table` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Eval run deleted successfully |  |
| `400` | Missing or multiple component filters |  |
| `404` | Eval run not found |  |


#### `GET /client/api/v1/health`

**Check API health status**

Returns the current health status of the v1 API

- **Auth:** `api-key`
- **operationId:** `health_1`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | API is healthy | [`ApiResponseHealthStatus`](#schema-apiresponsehealthstatus) |
| `401` | Invalid API key | [`ApiResponseHealthStatus`](#schema-apiresponsehealthstatus) |
| `429` | Rate limit exceeded | [`ApiResponseHealthStatus`](#schema-apiresponsehealthstatus) |


#### `GET /client/api/v1/http-requests`

**List HTTP requests**

List all HTTP requests for the specified project with pagination

- **Auth:** `api-key`
- **operationId:** `listHttpRequests`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Project reference ID (preferred, e.g. proj_abc123) |
| `project_id` | query |  | integer (int64) | Numeric project ID (deprecated — use project_ref_id) |
| `page` | query |  | integer (int32) |  |
| `size` | query |  | integer (int32) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP requests retrieved successfully | [`ApiResponseListHttpRequestResponse`](#schema-apiresponselisthttprequestresponse) |
| `401` | Invalid API key | [`ApiResponseListHttpRequestResponse`](#schema-apiresponselisthttprequestresponse) |
| `403` | Access denied to project | [`ApiResponseListHttpRequestResponse`](#schema-apiresponselisthttprequestresponse) |


#### `POST /client/api/v1/http-requests`

**Create HTTP request**

Create a new HTTP request configuration for use by AI agents

- **Auth:** `api-key`
- **operationId:** `createHttpRequest`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Project reference ID (preferred, e.g. proj_abc123) |
| `project_id` | query |  | integer (int64) | Numeric project ID (deprecated — use project_ref_id) |

**Request body** (`application/json`)

Schema: [`HttpRequestCreateRequest`](#schema-httprequestcreaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object | yes |  |  |
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object | yes |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | HTTP request created successfully | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `400` | Invalid input or validation failed | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `401` | Invalid API key | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `403` | Access denied to project | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `404` | Organization or project not found | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |


#### `POST /client/api/v1/http-requests/test`

**Test HTTP request**

Test an HTTP request configuration to validate endpoint connectivity

- **Auth:** `api-key`
- **operationId:** `testHttpRequest`

**Request body** (`application/json`)

Schema: [`HttpRequestTestData`](#schema-httprequesttestdata)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes |  |  |
| `headers` | object | yes |  |  |
| `data` | object |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request test completed | [`ApiResponseHttpRequestTestResponse`](#schema-apiresponsehttprequesttestresponse) |
| `400` | Invalid input or validation failed | [`ApiResponseHttpRequestTestResponse`](#schema-apiresponsehttprequesttestresponse) |
| `401` | Invalid API key | [`ApiResponseHttpRequestTestResponse`](#schema-apiresponsehttprequesttestresponse) |


#### `GET /client/api/v1/http-requests/{idOrRef}`

**Get HTTP request**

Retrieve a specific HTTP request configuration by ID

- **Auth:** `api-key`
- **operationId:** `getHttpRequest`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | HTTP request UUID (requestId, preferred) or numeric ID (deprecated) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request retrieved successfully | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `401` | Invalid API key | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `403` | Access denied | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `404` | HTTP request not found | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |


#### `PUT /client/api/v1/http-requests/{idOrRef}`

**Update HTTP request**

Update an existing HTTP request configuration

- **Auth:** `api-key`
- **operationId:** `updateHttpRequest`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | HTTP request UUID (requestId, preferred) or numeric ID (deprecated) |

**Request body** (`application/json`)

Schema: [`HttpRequestUpdateRequest`](#schema-httprequestupdaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object |  |  |  |
| `httpMethod` | string |  | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string |  | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object |  |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | HTTP request updated successfully | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `400` | Invalid input or validation failed | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `401` | Invalid API key | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `403` | Access denied | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |
| `404` | HTTP request not found | [`ApiResponseHttpRequestResponse`](#schema-apiresponsehttprequestresponse) |


#### `DELETE /client/api/v1/http-requests/{idOrRef}`

**Delete HTTP request**

Soft delete an HTTP request configuration. Cannot delete if agents reference it

- **Auth:** `api-key`
- **operationId:** `deleteHttpRequest`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | HTTP request UUID (requestId, preferred) or numeric ID (deprecated) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | HTTP request deleted successfully |  |
| `401` | Invalid API key |  |
| `403` | Access denied |  |
| `404` | HTTP request not found |  |
| `409` | Conflict - HTTP request is referenced by one or more agents |  |


#### `GET /client/api/v1/logs`

**Get agent run logs**

Returns paginated list of agent execution logs with filtering. Sortable fields: created/created_ts (default, descending), id, inputTokens/input_tokens, outputTokens/output_tokens, duration/duration_seconds. Date range is inclusive (full days). Maximum page size: 100.

- **Auth:** `api-key`
- **operationId:** `getAgentRunLogs`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `endDate` | query | yes | string (date) | End date (inclusive) |
| `startDate` | query | yes | string (date) | Start date (inclusive) |
| `status` | query |  | string | Filter by execution status |
| `scope` | query |  | string | Filter by usage scope |
| `usageMode` | query |  | string | Filter by usage mode |
| `tags` | query |  | string[] | Filter by tag names (preferred) |
| `tagIds` | query |  | integer (int64)[] | Filter by tag IDs (deprecated, use tags) |
| `teamRefIds` | query |  | string[] | Filter by team reference IDs (referenceId from usage response) |
| `teamIds` | query |  | integer (int64)[] | Filter by team IDs (deprecated, use teamRefIds) |
| `agentRefIds` | query |  | string[] | Filter by agent reference IDs (referenceId from usage response) |
| `agentIds` | query |  | integer (int64)[] | Filter by agent IDs (deprecated, use agentRefIds) |
| `projectIds` | query |  | integer (int64)[] | Filter by project numeric IDs (deprecated, use projectRefIds) |
| `projectRefIds` | query |  | string[] | Filter by project reference IDs (e.g. proj_...) |
| `workflowIds` | query |  | string[] | Filter by workflow IDs |
| `userRefIds` | query |  | string[] | Filter by user reference IDs (referenceId from usage response) |
| `userIds` | query |  | integer (int64)[] | Filter by user IDs (deprecated, use userRefIds) |
| `apiKeyRefIds` | query |  | string[] | Filter by API key reference IDs (UUID format) |
| `apiKeyIds` | query |  | integer (int64)[] | Filter by API key IDs (deprecated, use apiKeyRefIds) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Logs retrieved successfully | [`PagedResponseAgentRunDto`](#schema-pagedresponseagentrundto) |
| `400` | Invalid request parameters | [`PagedResponseAgentRunDto`](#schema-pagedresponseagentrundto) |
| `401` | Invalid API key | [`PagedResponseAgentRunDto`](#schema-pagedresponseagentrundto) |
| `429` | Rate limit exceeded | [`PagedResponseAgentRunDto`](#schema-pagedresponseagentrundto) |


#### `GET /client/api/v1/logs/by-run-id/{runId}`

**Get agent run log details by run ID**

Returns detailed execution data for a specific log by its string run ID (AI service identifier)

- **Auth:** `api-key`
- **operationId:** `getAgentRunDetailByRunId`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `runId` | path | yes | string | String run ID from AI service |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Log detail retrieved successfully | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `401` | Invalid API key | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `404` | Log not found | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `429` | Rate limit exceeded | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |


#### `GET /client/api/v1/logs/{id}`

**Get agent run log details by numeric ID (deprecated)**

Returns detailed execution data for a specific log by numeric DB ID. Deprecated: use GET /by-run-id/{runId} instead.

- **Auth:** `api-key`
- **operationId:** `getAgentRunDetailById`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `id` | path | yes | integer (int64) | Numeric log ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Log detail retrieved successfully | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `401` | Invalid API key | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `404` | Log not found | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |
| `429` | Rate limit exceeded | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |


#### `GET /client/api/v1/organizations/limits`

**Get organization limits**

Returns the active rate limits and file upload limits for the organization associated with the API key

- **Auth:** `api-key`
- **operationId:** `getOrganizationLimits`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Limits retrieved successfully | [`ApiResponseOrganizationLimitsDto`](#schema-apiresponseorganizationlimitsdto) |
| `401` | Invalid API key | [`ApiResponseOrganizationLimitsDto`](#schema-apiresponseorganizationlimitsdto) |


#### `GET /client/api/v1/projects/{projectRefId}/keys`

**List API keys**

Returns all API keys for the project, paginated

- **Auth:** `api-key`
- **operationId:** `listApiKeys`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |
| `isActive` | query |  | boolean |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PagedResponseApiKeyDto`](#schema-pagedresponseapikeydto) |


#### `POST /client/api/v1/projects/{projectRefId}/keys`

**Create API key**

Creates a new API key for the project

- **Auth:** `api-key`
- **operationId:** `createApiKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`ApiKeyV1Request`](#schema-apikeyv1request)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseApiKeyDto`](#schema-apiresponseapikeydto) |


#### `GET /client/api/v1/projects/{projectRefId}/keys/{keyRefId}`

**Get API key**

Returns a single API key by reference ID

- **Auth:** `api-key`
- **operationId:** `getApiKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |
| `keyRefId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseApiKeyDto`](#schema-apiresponseapikeydto) |


#### `PUT /client/api/v1/projects/{projectRefId}/keys/{keyRefId}`

**Update API key**

Updates an API key's name

- **Auth:** `api-key`
- **operationId:** `updateApiKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |
| `keyRefId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`ApiKeyV1Request`](#schema-apikeyv1request)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseApiKeyDto`](#schema-apiresponseapikeydto) |


#### `DELETE /client/api/v1/projects/{projectRefId}/keys/{keyRefId}`

**Revoke API key**

Revokes an API key

- **Auth:** `api-key`
- **operationId:** `revokeApiKey`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectRefId` | path | yes | string |  |
| `keyRefId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseUnit`](#schema-apiresponseunit) |


#### `GET /client/api/v1/skills`

**List skills**

Returns paginated skills accessible by the API key, optionally filtered by project and name. Use query parameters: page (0-indexed), size (default 20), sort (e.g. 'name,asc'), name (optional filter by name, case-insensitive partial match)

- **Auth:** `api-key`
- **operationId:** `listSkills`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |
| `name` | query |  | string | Filter by skill name (case-insensitive partial match) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skills retrieved successfully | [`PagedResponseSkillResponse`](#schema-pagedresponseskillresponse) |
| `401` | Invalid API key | [`PagedResponseSkillResponse`](#schema-pagedresponseskillresponse) |
| `429` | Rate limit exceeded | [`PagedResponseSkillResponse`](#schema-pagedresponseskillresponse) |


#### `POST /client/api/v1/skills`

**Create a skill**

- **Auth:** `api-key`
- **operationId:** `createSkill`

**Request body** (`application/json`)

Schema: [`SkillV1CreateRequest`](#schema-skillv1createrequest)

> Request to create a skill via the v1 API (includes projectId)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `projectRefId` | string | yes |  |  |
| `name` | string | yes | minLength=`2` ; maxLength=`64` ; pattern=`^[a-z0-9][a-z0-9-]*[a-z0-9]$` |  |
| `description` | string | yes | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Skill created successfully | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `400` | Invalid request data | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `401` | Invalid API key | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `503` | AI service unavailable | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |


#### `GET /client/api/v1/skills/{skillId}`

**Get skill by ID**

- **Auth:** `api-key`
- **operationId:** `getSkill`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `skillId` | path | yes | string | AI service skill ID (UUID) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill found | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `401` | Invalid API key | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `404` | Skill not found | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `429` | Rate limit exceeded | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |


#### `PATCH /client/api/v1/skills/{skillId}`

**Update a skill**

- **Auth:** `api-key`
- **operationId:** `updateSkill`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `skillId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`SkillUpdateRequest`](#schema-skillupdaterequest)

> Request to partially update an existing skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string |  | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |
| `isActive` | boolean |  |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill updated successfully | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `400` | Invalid request data | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `401` | Invalid API key | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `404` | Skill not found | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `503` | AI service unavailable | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |


#### `DELETE /client/api/v1/skills/{skillId}`

**Soft-delete a skill**

- **Auth:** `api-key`
- **operationId:** `deleteSkill`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `skillId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Skill deleted successfully |  |
| `401` | Invalid API key |  |
| `404` | Skill not found |  |


#### `POST /client/api/v1/skills/{skillId}/activate`

**Reactivate a soft-deleted skill**

- **Auth:** `api-key`
- **operationId:** `activateSkill`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `skillId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Skill activated successfully | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `401` | Invalid API key | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |
| `404` | Skill not found | [`ApiResponseSkillResponse`](#schema-apiresponseskillresponse) |


#### `POST /client/api/v1/skills/{skillId}/execute`

**Execute a skill script in a sandbox**

- **Auth:** `api-key`
- **operationId:** `executeSkill`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `skillId` | path | yes | string |  |

**Request body** (`application/json`)

Schema: [`SkillExecuteRequest`](#schema-skillexecuterequest)

> Request to execute a skill script

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `scriptName` | string | yes | minLength=`1` |  |
| `args` | string[] |  |  |  |
| `timeout` | integer (int32) |  |  | Execution timeout in seconds (1–600) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Script executed successfully | [`ApiResponseSkillExecuteResponse`](#schema-apiresponseskillexecuteresponse) |
| `400` | Script not found in skill | [`ApiResponseSkillExecuteResponse`](#schema-apiresponseskillexecuteresponse) |
| `401` | Invalid API key | [`ApiResponseSkillExecuteResponse`](#schema-apiresponseskillexecuteresponse) |
| `404` | Skill not found | [`ApiResponseSkillExecuteResponse`](#schema-apiresponseskillexecuteresponse) |
| `503` | Sandbox execution service unreachable | [`ApiResponseSkillExecuteResponse`](#schema-apiresponseskillexecuteresponse) |


#### `GET /client/api/v1/storage-resources`

**List storage resources**

Lists all storage resources for all projects accessible by the API key

- **Auth:** `api-key`
- **operationId:** `listStorageResources`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resources retrieved successfully | [`ApiResponseListStorageResourceResponse`](#schema-apiresponseliststorageresourceresponse) |
| `401` | Invalid API key | [`ApiResponseListStorageResourceResponse`](#schema-apiresponseliststorageresourceresponse) |


#### `POST /client/api/v1/storage-resources`

**Create a new storage resource**

Creates a new vector store for the specified project

- **Auth:** `api-key`
- **operationId:** `createStorageResource`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `project_id` | query |  | integer (int64) | Project numeric ID (deprecated, use project_ref_id) |
| `project_ref_id` | query |  | string | Project reference ID (e.g. proj_...) |

**Request body** (`application/json`)

Schema: [`StorageResourceCreateRequest`](#schema-storageresourcecreaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` | Name of the storage resource |
| `llmProviderRefId` | string |  |  | Reference ID (UUID) of the LLM provider configuration to use for embeddings. Obtain from GET /storage-resources/llm-providers (providerRefId field). Preferred over llmConfigId. |
| `llmConfigId` | integer (int64) |  |  | Internal numeric ID of the LLM provider configuration. Deprecated — use llmProviderRefId instead. |
| `embeddingModelId` | string | yes | minLength=`1` | ID of the embedding model to use |
| `metadataSchema` | [`MetadataSchemaField`](#schema-metadataschemafield)[] |  |  | Schema definition for metadata fields used to filter documents during search/retrieval |
| `invertedIndexConfig` | object |  |  | BM25 and inverted index tuning. bm25_b and bm25_k1 must be set together. See https://docs.weaviate.io/weaviate/config-refs/indexing/inverted-index |
| `vectorIndexType` | string |  | `hnsw`, `flat`, `dynamic`, `HNSW`, `FLAT`, `DYNAMIC` | Vector index type: hnsw (default), flat, or dynamic. |
| `vectorIndexConfig` | object |  |  | Vector index tuning. Available keys depend on vector_index_type (HNSW, FLAT, DYNAMIC). See https://docs.weaviate.io/weaviate/config-refs/indexing/vector-index |
| `searchConfig` | object |  |  | Search behavior configuration. Allows setting the search type (vector, keyword, or hybrid) and hybrid_search_alpha (0.0–1.0, must be 0.0 for keyword). |
| `isLlmProviderSpecified` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Storage resource created successfully | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `400` | Invalid request parameters | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `401` | Invalid API key | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `403` | Access denied to project | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `503` | Storage service unavailable | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |


#### `GET /client/api/v1/storage-resources/llm-providers`

**Get configured LLM providers with embedding models**

Lists LLM provider configurations with available embedding models for creating vector stores

- **Auth:** `api-key`
- **operationId:** `getConfiguredLlmProviders`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Embedding providers retrieved successfully | [`ApiResponseListLlmProviderConfigWithModelsResponse`](#schema-apiresponselistllmproviderconfigwithmodelsresponse) |
| `401` | Invalid API key | [`ApiResponseListLlmProviderConfigWithModelsResponse`](#schema-apiresponselistllmproviderconfigwithmodelsresponse) |
| `403` | Access denied | [`ApiResponseListLlmProviderConfigWithModelsResponse`](#schema-apiresponselistllmproviderconfigwithmodelsresponse) |


#### `GET /client/api/v1/storage-resources/{vectorStoreId}`

**Get storage resource by ID**

Retrieves detailed information about a specific storage resource

- **Auth:** `api-key`
- **operationId:** `getStorageResource`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resource retrieved successfully | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `401` | Invalid API key | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `403` | Access denied | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `404` | Storage resource not found | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |


#### `PUT /client/api/v1/storage-resources/{vectorStoreId}`

**Update storage resource**

Updates the name of a storage resource

- **Auth:** `api-key`
- **operationId:** `updateStorageResource`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) |  |

**Request body** (`application/json`)

Schema: [`StorageResourceUpdateRequest`](#schema-storageresourceupdaterequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Storage resource retrieved successfully | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `401` | Invalid API key | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `403` | Access denied | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |
| `404` | Storage resource not found | [`ApiResponseStorageResourceResponse`](#schema-apiresponsestorageresourceresponse) |


#### `DELETE /client/api/v1/storage-resources/{vectorStoreId}`

**Delete a storage resource**

Deletes a storage resource (vector store)

- **Auth:** `api-key`
- **operationId:** `deleteStorageResource`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Storage resource deleted successfully |  |
| `401` | Invalid API key |  |
| `403` | Access denied |  |
| `404` | Storage resource not found |  |
| `409` | Storage resource is in use by agents or teams |  |


#### `POST /client/api/v1/storage-resources/{vectorStoreId}/crawl`

**Crawl URLs**

Initiates web crawling for one or more URLs and adds content to the vector store

- **Auth:** `api-key`
- **operationId:** `crawlUrl`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |

**Request body** (`application/json`)

Schema: [`CrawlRequest`](#schema-crawlrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `urls` | [`CrawlUrlRequest`](#schema-crawlurlrequest)[] | yes |  |  |
| `ignoreLinks` | boolean | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Crawl task initiated successfully | [`ApiResponseCrawlResultDto`](#schema-apiresponsecrawlresultdto) |
| `401` | Invalid API key | [`ApiResponseCrawlResultDto`](#schema-apiresponsecrawlresultdto) |
| `403` | Access denied | [`ApiResponseCrawlResultDto`](#schema-apiresponsecrawlresultdto) |
| `404` | Storage resource not found | [`ApiResponseCrawlResultDto`](#schema-apiresponsecrawlresultdto) |


#### `GET /client/api/v1/storage-resources/{vectorStoreId}/files`

**List files in storage resource**

Lists files in the vector store with pagination

- **Auth:** `api-key`
- **operationId:** `listFiles`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |
| `page` | query |  | integer (int32) | Page number (0-based) |
| `pageSize` | query |  | integer (int32) | Page size |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Files retrieved successfully | [`ApiResponseFileListDto`](#schema-apiresponsefilelistdto) |
| `401` | Invalid API key | [`ApiResponseFileListDto`](#schema-apiresponsefilelistdto) |
| `403` | Access denied | [`ApiResponseFileListDto`](#schema-apiresponsefilelistdto) |
| `404` | Storage resource not found | [`ApiResponseFileListDto`](#schema-apiresponsefilelistdto) |


#### `POST /client/api/v1/storage-resources/{vectorStoreId}/files`

**Upload files to storage resource**

Uploads files to the vector store. Files are processed asynchronously for embedding. Optionally specify metadata for filtering.

- **Auth:** `api-key`
- **operationId:** `uploadFiles`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |
| `chunkingStrategy` | query |  | string |  |
| `chunkingConfig` | query |  | string |  |
| `metadata` | query |  | string |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "description": "Files to upload",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  },
  "required": [
    "files"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | File upload started successfully | [`ApiResponseFileUploadResultDto`](#schema-apiresponsefileuploadresultdto) |
| `400` | Invalid request - no files, empty files, or invalid JSON | [`ApiResponseFileUploadResultDto`](#schema-apiresponsefileuploadresultdto) |
| `401` | Invalid API key | [`ApiResponseFileUploadResultDto`](#schema-apiresponsefileuploadresultdto) |
| `403` | Access denied | [`ApiResponseFileUploadResultDto`](#schema-apiresponsefileuploadresultdto) |
| `404` | Storage resource not found | [`ApiResponseFileUploadResultDto`](#schema-apiresponsefileuploadresultdto) |


#### `DELETE /client/api/v1/storage-resources/{vectorStoreId}/files/{fileId}`

**Delete a file from storage resource**

Deletes a file from the vector store

- **Auth:** `api-key`
- **operationId:** `deleteFile`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) | Vector store ID |
| `fileId` | path | yes | string | File ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | File deleted successfully |  |
| `401` | Invalid API key |  |
| `403` | Access denied |  |
| `404` | Storage resource or file not found |  |


#### `GET /client/api/v1/storage-resources/{vectorStoreId}/files/{fileId}/download`

**Download file**

Downloads the original file from the storage resource

- **Auth:** `api-key`
- **operationId:** `downloadFile`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `vectorStoreId` | path | yes | string (uuid) |  |
| `fileId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | File downloaded successfully | `string` |
| `401` | Invalid API key | `string` |
| `403` | Access denied | `string` |
| `404` | Storage resource or file not found | `string` |


#### `GET /client/api/v1/tags`

**Get organization tags**

Returns all tags for an organization

- **Auth:** `api-key`
- **operationId:** `getOrganizationTags`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseListSimpleTagDto`](#schema-apiresponselistsimpletagdto) |


#### `GET /client/api/v1/teams`

**List available teams**

Returns a list of all available AI teams with their metadata and configuration

- **Auth:** `api-key`
- **operationId:** `listTeams`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `name` | query |  | string |  |
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Teams retrieved successfully | [`ApiResponseListTeamResponse`](#schema-apiresponselistteamresponse) |
| `401` | Invalid API key | [`ApiResponseListTeamResponse`](#schema-apiresponselistteamresponse) |


#### `POST /client/api/v1/teams`

**Create a new team**

Creates a new AI team with multi-agent configuration. The team is created both in the database and in the AI service.

- **Auth:** `api-key`
- **operationId:** `createTeam`

**Request body** (`application/json`)

Schema: [`TeamV1CreateRequest`](#schema-teamv1createrequest)

> Request to create a new team via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name known to the AI (e.g. 'Support Team') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Support') |
| `projectRefId` | string |  |  | Reference ID of the project this team belongs to (e.g. 'proj_...'). Use this instead of projectId. |
| `projectId` | integer (int64) |  |  | Numeric ID of the project. Deprecated — use projectRefId. |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) | yes |  | Team configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Team created successfully | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `400` | Invalid request data or validation failed | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `401` | Invalid API key | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `503` | AI service unavailable | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |


#### `GET /client/api/v1/teams/jobs/{jobId}/status`

**Get team job status**

Returns the status, progress, and results of an asynchronous team execution job.

- **Auth:** `api-key`
- **operationId:** `getJobStatus`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `jobId` | path | yes | string |  |
| `include_messages` | query |  | boolean | Include full message history in response (default: false for backward compatibility and performance) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Job status retrieved successfully | [`TeamJobResponseDto`](#schema-teamjobresponsedto) |
| `401` | Invalid API key | [`TeamJobResponseDto`](#schema-teamjobresponsedto) |
| `404` | Job not found | [`TeamJobResponseDto`](#schema-teamjobresponsedto) |


#### `GET /client/api/v1/teams/options`

**listTeamOptions**

- **Auth:** `api-key`
- **operationId:** `listTeamOptions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `name` | query |  | string |  |
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`TeamOptionV1Dto`](#schema-teamoptionv1dto)[] |


#### `GET /client/api/v1/teams/{teamId}`

**Get team by ID**

Retrieves team details by AI service team ID

- **Auth:** `api-key`
- **operationId:** `getTeam`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team found | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `401` | Invalid API key | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `404` | Team not found | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |


#### `PUT /client/api/v1/teams/{teamId}`

**Update a team**

Updates an existing team's name, description, and/or configuration

- **Auth:** `api-key`
- **operationId:** `updateTeam`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`application/json`)

Schema: [`TeamV1UpdateRequest`](#schema-teamv1updaterequest)

> Request to update an existing team via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name (optional - only updates if provided) |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description (optional - only updates if provided) |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) |  |  | Team configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team updated successfully | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `400` | Invalid request data | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `401` | Invalid API key | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `404` | Team not found | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `503` | AI service unavailable | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |


#### `DELETE /client/api/v1/teams/{teamId}`

**Delete a team**

Soft deletes a team. The team is marked as deleted in the database and deletion is attempted in the AI service.

- **Auth:** `api-key`
- **operationId:** `deleteTeam`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Team deleted successfully |  |
| `401` | Invalid API key |  |
| `404` | Team not found |  |


#### `POST /client/api/v1/teams/{teamId}/clone`

**Clone a team**

Creates a copy of an existing team with a new name

- **Auth:** `api-key`
- **operationId:** `cloneTeam`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |

**Request body** (`application/json`)

Schema: [`TeamCloneRequest`](#schema-teamclonerequest)

> Request to clone a team

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned team |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name for the cloned team shown in the platform dashboard |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Team cloned successfully | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `400` | Invalid request data | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `401` | Invalid API key | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `404` | Team not found | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |
| `503` | AI service unavailable | [`ApiResponseTeamResponse`](#schema-apiresponseteamresponse) |


#### `POST /client/api/v1/teams/{teamId}/run`

**Execute team (non-streaming)**

Executes a team and returns the complete response with metrics. Usage is automatically tracked.

- **Auth:** `api-key`
- **operationId:** `executeTeam`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |
| `message` | query | yes | string | User message/prompt for the team |
| `tag` | query |  | string |  |
| `session_id` | query |  | string |  |
| `user_id` | query |  | string | External user identifier (UUID). If not provided, a new one will be generated. |
| `monitor` | query |  | boolean | Enable monitoring (default: true) |
| `include_messages` | query |  | boolean | Include full message history in response (default: false). Set to true to include detailed conversation steps and tool calls. |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team executed successfully | [`ApiResponseTeamRunResponseData`](#schema-apiresponseteamrunresponsedata) |
| `400` | Invalid request data | [`ApiResponseTeamRunResponseData`](#schema-apiresponseteamrunresponsedata) |
| `401` | Invalid API key | [`ApiResponseTeamRunResponseData`](#schema-apiresponseteamrunresponsedata) |
| `404` | Team not found | [`ApiResponseTeamRunResponseData`](#schema-apiresponseteamrunresponsedata) |
| `503` | AI service unavailable | [`ApiResponseTeamRunResponseData`](#schema-apiresponseteamrunresponsedata) |


#### `POST /client/api/v1/teams/{teamId}/run-async`

**Execute team asynchronously**

Starts asynchronous execution of the specified team and returns a job ID for status tracking

- **Auth:** `api-key`
- **operationId:** `executeTeamAsync`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |
| `message` | query | yes | string |  |
| `tag` | query |  | string |  |
| `session_id` | query |  | string |  |
| `user_id` | query |  | string | External user identifier (UUID). If not provided, a new one will be generated. |
| `monitor` | query |  | boolean | Enable monitoring (default: true) |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `202` | Team execution started | [`TeamJobCreatedDto`](#schema-teamjobcreateddto) |
| `400` | Invalid request data | [`TeamJobCreatedDto`](#schema-teamjobcreateddto) |
| `401` | Invalid API key | [`TeamJobCreatedDto`](#schema-teamjobcreateddto) |
| `404` | Team not found | [`TeamJobCreatedDto`](#schema-teamjobcreateddto) |
| `503` | AI service unavailable | [`TeamJobCreatedDto`](#schema-teamjobcreateddto) |


#### `POST /client/api/v1/teams/{teamId}/run-stream`

**Execute team (streaming)**

Executes a team and streams the response via Server-Sent Events (SSE). Usage is tracked when metrics are received.

- **Auth:** `api-key`
- **operationId:** `executeTeamStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | AI service team ID |
| `message` | query | yes | string | User message/prompt for the team |
| `tag` | query |  | string |  |
| `session_id` | query |  | string |  |
| `user_id` | query |  | string | External user identifier (UUID). If not provided, a new one will be generated. |
| `monitor` | query |  | boolean | Enable monitoring (default: true) |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "files": {
      "type": "array",
      "description": "Optional file attachments (multipart)",
      "items": {
        "type": "string",
        "format": "binary"
      }
    }
  }
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Team execution stream started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid request data | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `401` | Invalid API key | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `404` | Team not found | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI service unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /client/api/v1/teams/{teamId}/runs/{runId}/cancel`

**Cancel a team run**

Cancels an in-progress team run. Usage is tracked after successful cancellation.

- **Auth:** `api-key`
- **operationId:** `cancelTeamRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string |  |
| `runId` | path | yes | string |  |
| `session_id` | query | yes | string |  |
| `user_id` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully | [`ApiResponseString`](#schema-apiresponsestring) |
| `401` | Invalid API key | [`ApiResponseString`](#schema-apiresponsestring) |
| `404` | Team or run not found | [`ApiResponseString`](#schema-apiresponsestring) |
| `503` | AI system unavailable | [`ApiResponseString`](#schema-apiresponsestring) |


#### `GET /client/api/v1/teams/{teamId}/sessions`

**List team sessions**

Returns a paginated list of sessions for the specified team.
            Sessions are filtered by the authenticated API key's organization and project.
            Optionally filter by session name.

- **Auth:** `api-key`
- **operationId:** `listSessions_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | Team ID |
| `user_id` | query | yes | string | Filter sessions by user ID |
| `sessionName` | query |  | string | Filter sessions by name (optional) |
| `limit` | query |  | integer (int32) | Number of results per page |
| `page` | query |  | integer (int32) | Page number (1-based) |
| `sortBy` | query |  | string | Sort field |
| `sortOrder` | query |  | string | Sort order (asc/desc) |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Sessions retrieved successfully | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `400` | Invalid request parameters | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `401` | Invalid or missing API key | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `404` | Team not found or doesn't belong to project | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |
| `429` | Rate limit exceeded | [`PagedResponseSessionListDto`](#schema-pagedresponsesessionlistdto) |


#### `GET /client/api/v1/teams/{teamId}/sessions/{sessionId}`

**Get team session by ID**

Retrieves detailed information about a specific session

- **Auth:** `api-key`
- **operationId:** `getSession_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | Team ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session retrieved successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Team or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `DELETE /client/api/v1/teams/{teamId}/sessions/{sessionId}`

**Delete team session**

Deletes a specific session for the team

- **Auth:** `api-key`
- **operationId:** `deleteSession_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | Team ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Session deleted successfully |  |
| `401` | Invalid or missing API key |  |
| `404` | Team or session not found |  |


#### `POST /client/api/v1/teams/{teamId}/sessions/{sessionId}/rename`

**Rename team session**

Updates the name of a specific session

- **Auth:** `api-key`
- **operationId:** `renameSession_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | Team ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Request body** (`application/json`)

Schema: [`RenameSessionRequestDto`](#schema-renamesessionrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session renamed successfully | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `400` | Invalid request data | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `401` | Invalid or missing API key | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |
| `404` | Team or session not found | [`ApiResponseSessionDto`](#schema-apiresponsesessiondto) |


#### `GET /client/api/v1/teams/{teamId}/sessions/{sessionId}/runs`

**Get team session runs**

Returns all runs (execution history) for a specific session

- **Auth:** `api-key`
- **operationId:** `getSessionRuns_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `teamId` | path | yes | string | Team ID |
| `sessionId` | path | yes | string | Session ID |
| `user_id` | query | yes | string | User ID |
| `dbId` | query |  | string | Optional database ID for multi-tenant scenarios |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Session runs retrieved successfully | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `401` | Invalid or missing API key | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |
| `404` | Team or session not found | [`ApiResponseListSessionRunDto`](#schema-apiresponselistsessionrundto) |


#### `GET /client/api/v1/telephony-presets`

**List telephony presets**

List all telephony presets with pagination and optional filtering by name and type

- **Auth:** `api-key`
- **operationId:** `listTelephonyPresets`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `name` | query |  | string | Filter by name (case-insensitive, partial match) |
| `type` | query |  | string | Filter by type (INBOUND or OUTBOUND) |
| `project_ref_id` | query |  | string | Filter by project reference ID (e.g. proj_...) |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PagedResponseTelephonyPresetResponseDto`](#schema-pagedresponsetelephonypresetresponsedto) |


#### `POST /client/api/v1/telephony-presets`

**Create telephony preset**

Create a new telephony preset with SIP trunk configuration

- **Auth:** `api-key`
- **operationId:** `createTelephonyPreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `projectId` | query |  | integer (int64) | Project numeric ID (deprecated, use projectRefId) |
| `projectRefId` | query |  | string | Project reference ID (e.g. proj_...) |

**Request body** (`application/json`)

Schema: [`CreateTelephonyPresetDto`](#schema-createtelephonypresetdto)

> Request DTO for creating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Created | [`ApiResponseTelephonyPresetResponseDto`](#schema-apiresponsetelephonypresetresponsedto) |


#### `GET /client/api/v1/telephony-presets/{idOrRef}`

**Get telephony preset**

Retrieve a telephony preset by numeric ID (deprecated) or string reference ID (e.g. telephony_...)

- **Auth:** `api-key`
- **operationId:** `getTelephonyPreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Telephony preset numeric ID or reference ID (e.g. telephony_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseTelephonyPresetResponseDto`](#schema-apiresponsetelephonypresetresponsedto) |


#### `PUT /client/api/v1/telephony-presets/{idOrRef}`

**Update telephony preset**

Update an existing telephony preset by numeric ID (deprecated) or string reference ID (e.g. telephony_...)

- **Auth:** `api-key`
- **operationId:** `updateTelephonyPreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Telephony preset numeric ID or reference ID (e.g. telephony_...) |

**Request body** (`application/json`)

Schema: [`UpdateTelephonyPresetDto`](#schema-updatetelephonypresetdto)

> Request DTO for updating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseTelephonyPresetResponseDto`](#schema-apiresponsetelephonypresetresponsedto) |


#### `DELETE /client/api/v1/telephony-presets/{idOrRef}`

**Delete telephony preset**

Delete a telephony preset by numeric ID (deprecated) or string reference ID (e.g. telephony_...)

- **Auth:** `api-key`
- **operationId:** `deleteTelephonyPreset`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Telephony preset numeric ID or reference ID (e.g. telephony_...) |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | No Content |  |


#### `POST /client/api/v1/telephony-presets/{idOrRef}/outbound-call`

**Create outbound call**

Initiate an outbound phone call using this telephony preset. Accepts numeric ID (deprecated) or string reference ID (e.g. telephony_...)

- **Auth:** `api-key`
- **operationId:** `createOutboundCall`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `idOrRef` | path | yes | string | Telephony preset numeric ID or reference ID (e.g. telephony_...) |
| `user_id` | query |  | string |  |
| `session_id` | query |  | string |  |

**Request body** (`application/json`)

Schema: [`CreateOutboundCallDto`](#schema-createoutboundcalldto)

> Request to initiate an outbound call

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `fromNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Your outbound phone number to call from (must be in telephony preset) |
| `toNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Phone number to call |
| `playDialtone` | boolean | yes |  | Play dial tone to room until call is answered |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `201` | Created | [`ApiResponseOutboundCallResponseDto`](#schema-apiresponseoutboundcallresponsedto) |


#### `GET /client/api/v1/traces`

**List traces**

Returns a paginated list of execution traces.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Filter traces for a specific agent
            - `teamId`: Filter traces for a specific team

            **Optional Filters**: user_id, run_id, session_id, status, start_time, end_time

- **Auth:** `api-key`
- **operationId:** `listTraces`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `userId` | query |  | string | Filter by user ID (optional) |
| `runId` | query |  | string | Filter by run ID |
| `sessionId` | query |  | string | Filter by session ID |
| `status` | query |  | string | Filter by status (OK, ERROR) |
| `startTime` | query |  | string | Filter traces after this time |
| `endTime` | query |  | string | Filter traces before this time |
| `page` | query |  | integer (int32) | Page number (1-based) |
| `limit` | query |  | integer (int32) | Number of results per page |
| `dbId` | query |  | string | Optional database ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Traces retrieved successfully | [`PagedResponseTraceListDto`](#schema-pagedresponsetracelistdto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`PagedResponseTraceListDto`](#schema-pagedresponsetracelistdto) |
| `401` | Invalid or missing API key | [`PagedResponseTraceListDto`](#schema-pagedresponsetracelistdto) |
| `404` | Agent/Team not found | [`PagedResponseTraceListDto`](#schema-pagedresponsetracelistdto) |


#### `GET /client/api/v1/traces/session-stats`

**Get trace session statistics**

Retrieves aggregated trace statistics grouped by session.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Statistics for this agent
            - `teamId`: Statistics for this team

            **Optional Filters**: user_id, start_time, end_time

- **Auth:** `api-key`
- **operationId:** `getTraceSessionStats`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `userId` | query |  | string | Filter by user ID (optional) |
| `startTime` | query |  | string | Filter after this time |
| `endTime` | query |  | string | Filter before this time |
| `page` | query |  | integer (int32) | Page number (1-based) |
| `limit` | query |  | integer (int32) | Number of results per page |
| `dbId` | query |  | string | Optional database ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Statistics retrieved successfully | [`PagedResponseTraceSessionStatsDto`](#schema-pagedresponsetracesessionstatsdto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`PagedResponseTraceSessionStatsDto`](#schema-pagedresponsetracesessionstatsdto) |


#### `GET /client/api/v1/traces/{traceId}`

**Get trace detail**

Retrieves detailed information about a specific trace.

            **Required Filters**: Exactly one of the following must be provided:
            - `agentId`: Trace belongs to this agent
            - `teamId`: Trace belongs to this team

- **Auth:** `api-key`
- **operationId:** `getTrace`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `traceId` | path | yes | string | Trace ID |
| `agentId` | query |  | string | Filter by agent ID |
| `teamId` | query |  | string | Filter by team ID |
| `spanId` | query |  | string | Optional span ID |
| `runId` | query |  | string | Optional run ID |
| `dbId` | query |  | string | Optional database ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Trace retrieved successfully | [`ApiResponseTraceDetailDto`](#schema-apiresponsetracedetaildto) |
| `400` | Invalid request parameters or missing/multiple component filters | [`ApiResponseTraceDetailDto`](#schema-apiresponsetracedetaildto) |
| `404` | Trace not found | [`ApiResponseTraceDetailDto`](#schema-apiresponsetracedetaildto) |


#### `GET /client/api/v1/usage`

**getApiUsage**

- **Auth:** `api-key`
- **operationId:** `getApiUsage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `scope` | query |  | string |  |
| `usageMode` | query |  | string |  |
| `tags` | query |  | string[] |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamRefIds` | query |  | string[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentRefIds` | query |  | string[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |
| `userRefIds` | query |  | string[] |  |
| `userIds` | query |  | integer (int64)[] |  |
| `apiKeyRefIds` | query |  | string[] |  |
| `apiKeyIds` | query |  | integer (int64)[] |  |
| `granularity` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseUsageResponseDto`](#schema-apiresponseusageresponsedto) |


#### `GET /client/api/v1/usage/export`

**exportUsage**

- **Auth:** `api-key`
- **operationId:** `exportUsage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `endDate` | query | yes | string (date) |  |
| `startDate` | query | yes | string (date) |  |
| `scope` | query |  | string |  |
| `usageMode` | query |  | string |  |
| `tags` | query |  | string[] |  |
| `tagIds` | query |  | integer (int64)[] |  |
| `teamRefIds` | query |  | string[] |  |
| `teamIds` | query |  | integer (int64)[] |  |
| `agentRefIds` | query |  | string[] |  |
| `agentIds` | query |  | integer (int64)[] |  |
| `workflowIds` | query |  | string[] |  |
| `userRefIds` | query |  | string[] |  |
| `userIds` | query |  | integer (int64)[] |  |
| `apiKeyRefIds` | query |  | string[] |  |
| `apiKeyIds` | query |  | integer (int64)[] |  |
| `granularity` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `GET /client/api/v1/version`

**Get API version information**

Returns version and build information for the v1 API

- **Auth:** `api-key`
- **operationId:** `version`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ApiResponseMapStringObject`](#schema-apiresponsemapstringobject) |


#### `GET /client/api/v1/workflows`

**List available workflows**

Returns a list of all available workflows with their metadata

- **Auth:** `api-key`
- **operationId:** `listWorkflows`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflows retrieved successfully | [`ApiResponseListAiWorkflowListItem`](#schema-apiresponselistaiworkflowlistitem) |
| `401` | Invalid API key | [`ApiResponseListAiWorkflowListItem`](#schema-apiresponselistaiworkflowlistitem) |
| `429` | Rate limit exceeded | [`ApiResponseListAiWorkflowListItem`](#schema-apiresponselistaiworkflowlistitem) |
| `503` | AI system unavailable | [`ApiResponseListAiWorkflowListItem`](#schema-apiresponselistaiworkflowlistitem) |


#### `GET /client/api/v1/workflows/{workflowId}`

**Get workflow details**

Returns detailed information about a specific workflow including its input schema

- **Auth:** `api-key`
- **operationId:** `getWorkflowDetails`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflow details retrieved successfully | [`ApiResponseAiWorkflowDetailsResponse`](#schema-apiresponseaiworkflowdetailsresponse) |
| `401` | Invalid API key | [`ApiResponseAiWorkflowDetailsResponse`](#schema-apiresponseaiworkflowdetailsresponse) |
| `404` | Workflow not found | [`ApiResponseAiWorkflowDetailsResponse`](#schema-apiresponseaiworkflowdetailsresponse) |
| `429` | Rate limit exceeded | [`ApiResponseAiWorkflowDetailsResponse`](#schema-apiresponseaiworkflowdetailsresponse) |
| `503` | AI system unavailable | [`ApiResponseAiWorkflowDetailsResponse`](#schema-apiresponseaiworkflowdetailsresponse) |


#### `POST /client/api/v1/workflows/{workflowId}/run`

**Execute workflow synchronously**

Executes the specified workflow with given parameters and returns the result immediately

- **Auth:** `api-key`
- **operationId:** `executeWorkflow`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string |  |
| `project_id` | query |  | integer (int64) |  |
| `project_ref_id` | query |  | string |  |
| `tag` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Workflow executed successfully | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `400` | Invalid parameters or workflow not found | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `401` | Invalid API key | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `429` | Rate limit exceeded | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |
| `503` | AI system unavailable | [`AiWorkflowRunResponse`](#schema-aiworkflowrunresponse) |


#### `POST /client/api/v1/workflows/{workflowId}/run-stream`

**Execute workflow with streaming response**

Executes the specified workflow with streaming response for real-time output

- **Auth:** `api-key`
- **operationId:** `executeWorkflowStream`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string |  |
| `project_id` | query |  | integer (int64) |  |
| `project_ref_id` | query |  | string |  |
| `tag` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | Streaming execution started | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `400` | Invalid parameters | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `401` | Invalid API key | [`ServerSentEventString`](#schema-serversenteventstring)[] |
| `503` | AI system unavailable | [`ServerSentEventString`](#schema-serversenteventstring)[] |


#### `POST /client/api/v1/workflows/{workflowId}/runs/{runId}/cancel`

**Cancel a workflow run**

Cancels an in-progress workflow run. Usage is tracked after successful cancellation.

- **Auth:** `api-key`
- **operationId:** `cancelWorkflowRun`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `workflowId` | path | yes | string |  |
| `runId` | path | yes | string |  |
| `project_id` | query |  | integer (int64) |  |
| `project_ref_id` | query |  | string |  |
| `user_id` | query | yes | string |  |
| `session_id` | query | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Run cancelled successfully |  |
| `401` | Invalid API key |  |
| `404` | Workflow or run not found |  |
| `503` | AI system unavailable |  |


---

<a id="dom-other"></a>

### Other

_40 operations._

#### ▸ Languages

#### `GET /api/languages`

**getAll_2**

- **Auth:** `bearer-jwt`
- **operationId:** `getAll_2`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LanguageDto`](#schema-languagedto)[] |


#### ▸ LLM Models

#### `GET /api/llms`

**getAll_1**

- **Auth:** `bearer-jwt`
- **operationId:** `getAll_1`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmDto`](#schema-llmdto)[] |


#### `GET /api/organizations/{organizationId}/llms/providers`

**Get supported AI service providers**

- **Auth:** `bearer-jwt`
- **operationId:** `getAiServiceProviders_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) | Organization ID |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `204` | Agent deleted successfully | [`AIProviderDto`](#schema-aiproviderdto)[] |
| `403` | User does not have permission to delete this agent | [`AIProviderDto`](#schema-aiproviderdto)[] |
| `404` | Agent not found | [`AIProviderDto`](#schema-aiproviderdto)[] |


#### ▸ LLM Providers

#### `GET /api/organizations/{organizationId}/llm-providers`

**listAll**

- **Auth:** `bearer-jwt`
- **operationId:** `listAll`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigDto`](#schema-llmproviderconfigdto)[] |


#### `POST /api/organizations/{organizationId}/llm-providers`

**create_2**

- **Auth:** `bearer-jwt`
- **operationId:** `create_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`LlmProviderConfigRequest`](#schema-llmproviderconfigrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `providerId` | string | yes |  |  |
| `providerName` | string | yes |  |  |
| `env` | object |  |  |  |
| `config` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigDto`](#schema-llmproviderconfigdto) |


#### `GET /api/organizations/{organizationId}/llm-providers/export`

**exportConfigurations**

- **Auth:** `bearer-jwt`
- **operationId:** `exportConfigurations`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `POST /api/organizations/{organizationId}/llm-providers/import`

**importConfigurations**

- **Auth:** `bearer-jwt`
- **operationId:** `importConfigurations`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`multipart/form-data`)

```json
{
  "type": "object",
  "properties": {
    "file": {
      "type": "string",
      "format": "binary"
    }
  },
  "required": [
    "file"
  ]
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ImportConfigurationsResponse`](#schema-importconfigurationsresponse) |


#### `POST /api/organizations/{organizationId}/llm-providers/models`

**getLlmProviderModels**

- **Auth:** `bearer-jwt`
- **operationId:** `getLlmProviderModels`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`LlmProviderEnvRequest`](#schema-llmproviderenvrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  |  |
| `env` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmModelDto`](#schema-llmmodeldto)[] |


#### `GET /api/organizations/{organizationId}/llm-providers/options`

**getProviderOptions**

- **Auth:** `bearer-jwt`
- **operationId:** `getProviderOptions`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderOptionDto`](#schema-llmprovideroptiondto)[] |


#### `GET /api/organizations/{organizationId}/llm-providers/page`

**list_2**

- **Auth:** `bearer-jwt`
- **operationId:** `list_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `name` | query |  | string |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageLlmProviderConfigDto`](#schema-pagellmproviderconfigdto) |


#### `POST /api/organizations/{organizationId}/llm-providers/test-connection`

**testConnectionUnsaved**

- **Auth:** `bearer-jwt`
- **operationId:** `testConnectionUnsaved`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`LlmProviderEnvRequest`](#schema-llmproviderenvrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  |  |
| `env` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`TestConnectionResponse`](#schema-testconnectionresponse) |


#### `GET /api/organizations/{organizationId}/llm-providers/{id}`

**get_1**

- **Auth:** `bearer-jwt`
- **operationId:** `get_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigDto`](#schema-llmproviderconfigdto) |


#### `PUT /api/organizations/{organizationId}/llm-providers/{id}`

**update_2**

- **Auth:** `bearer-jwt`
- **operationId:** `update_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`LlmProviderConfigRequest`](#schema-llmproviderconfigrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `providerId` | string | yes |  |  |
| `providerName` | string | yes |  |  |
| `env` | object |  |  |  |
| `config` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigDto`](#schema-llmproviderconfigdto) |


#### `DELETE /api/organizations/{organizationId}/llm-providers/{id}`

**delete_2**

- **Auth:** `bearer-jwt`
- **operationId:** `delete_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `GET /api/organizations/{organizationId}/llm-providers/{id}/export`

**exportSingleConfiguration**

- **Auth:** `bearer-jwt`
- **operationId:** `exportSingleConfiguration`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `string` |


#### `GET /api/organizations/{organizationId}/llm-providers/{id}/models`

**getLlmProviderModelsForConfig**

- **Auth:** `bearer-jwt`
- **operationId:** `getLlmProviderModelsForConfig`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmModelDto`](#schema-llmmodeldto)[] |


#### `GET /api/organizations/{organizationId}/llm-providers/{id}/models/{modelId}/reasoning-params`

**getModelReasoningParams**

- **Auth:** `bearer-jwt`
- **operationId:** `getModelReasoningParams`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |
| `modelId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`ReasoningParamsResponse`](#schema-reasoningparamsresponse) |


#### `POST /api/organizations/{organizationId}/llm-providers/{id}/test-connection`

**testConnectionSaved**

- **Auth:** `bearer-jwt`
- **operationId:** `testConnectionSaved`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`TestConnectionResponse`](#schema-testconnectionresponse) |


#### `GET /api/organizations/{organizationId}/llm-providers/{id}/usage`

**getUsage**

- **Auth:** `bearer-jwt`
- **operationId:** `getUsage`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`LlmProviderConfigUsageDto`](#schema-llmproviderconfigusagedto) |


#### ▸ Audio Agents

#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents`

**list_1**

- **Auth:** `bearer-jwt`
- **operationId:** `list_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `name` | query |  | string |  |
| `pageable` | query | yes | [`Pageable`](#schema-pageable) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`PageVoicePresetSummaryDto`](#schema-pagevoicepresetsummarydto) |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/audio-agents`

**create_1**

- **Auth:** `bearer-jwt`
- **operationId:** `create_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`VoicePresetRequestDto`](#schema-voicepresetrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePresetDto`](#schema-voicepresetdto) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/mode`

**Get voice mode**

Returns the configured voice mode (LIVEKIT or WEBRTC)

- **Auth:** `bearer-jwt`
- **operationId:** `getVoiceMode_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceModeResponse`](#schema-voicemoderesponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/options`

**Get voice preset options**

Returns a list of all voice presets for the project as simple options (id, referenceId, name)

- **Auth:** `bearer-jwt`
- **operationId:** `getOptions_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePresetOptionDto`](#schema-voicepresetoptiondto)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}`

**get**

- **Auth:** `bearer-jwt`
- **operationId:** `get`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePresetDto`](#schema-voicepresetdto) |


#### `PUT /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}`

**update_1**

- **Auth:** `bearer-jwt`
- **operationId:** `update_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

Schema: [`VoicePresetRequestDto`](#schema-voicepresetrequestdto)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePresetDto`](#schema-voicepresetdto) |


#### `DELETE /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}`

**delete_1**

- **Auth:** `bearer-jwt`
- **operationId:** `delete_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK |  |


#### `POST /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/start-session`

**startVoiceSession_1**

- **Auth:** `bearer-jwt`
- **operationId:** `startVoiceSession_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |
| `session_id` | query |  | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceSessionResponse`](#schema-voicesessionresponse) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/stt/test-credentials`

**Test STT credentials**

Tests the STT (Speech-to-Text) credentials by making a test API call

- **Auth:** `bearer-jwt`
- **operationId:** `testSttCredentials_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/tts/test-credentials`

**Test TTS credentials**

Tests the TTS (Text-to-Speech) credentials by making a test API call

- **Auth:** `bearer-jwt`
- **operationId:** `testTtsCredentials_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/voices`

**getVoices_1**

- **Auth:** `bearer-jwt`
- **operationId:** `getVoices_1`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`Voice`](#schema-voice)[] |


#### `GET /api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/voices/{voiceId}/preview`

**Get voice preview**

Returns a presigned URL for a voice preview audio file

- **Auth:** `bearer-jwt`
- **operationId:** `getVoicePreview_2`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `organizationId` | path | yes | integer (int64) |  |
| `projectId` | path | yes | integer (int64) |  |
| `id` | path | yes | integer (int64) |  |
| `voiceId` | path | yes | string |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePreviewResponse`](#schema-voicepreviewresponse) |


#### ▸ vault-example-controller

#### `GET /api/vault/health`

**checkHealth**

- **Auth:** **Public** — no authentication required
- **operationId:** `checkHealth`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | `object` |


#### ▸ Voice Providers

#### `GET /api/voice-providers`

**getAll**

- **Auth:** `bearer-jwt`
- **operationId:** `getAll`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceProviderDto`](#schema-voiceproviderdto)[] |


#### `DELETE /api/voice-providers/cache/clear`

**clearPreviewCache**

- **Auth:** `bearer-jwt`
- **operationId:** `clearPreviewCache`

**Request body** (`application/json`)

Schema: [`VoiceRequest`](#schema-voicerequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKey` | string | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`DeletedFilesResponse`](#schema-deletedfilesresponse) |


#### `POST /api/voice-providers/stt/test-credentials`

**testApiKey_1**

- **Auth:** `bearer-jwt`
- **operationId:** `testApiKey_1`

**Request body** (`application/json`)

Schema: [`STTProviderTestRequest`](#schema-sttprovidertestrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes | `google`, `soniox`, `azure`, `cartesia`, `eleven`, `deepgram`, `assemblyai` |  |
| `credentials` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `POST /api/voice-providers/tts/test-credentials`

**testApiKey**

- **Auth:** `bearer-jwt`
- **operationId:** `testApiKey`

**Request body** (`application/json`)

Schema: [`TTSProviderTestRequest`](#schema-ttsprovidertestrequest)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes | `azure`, `openai`, `cartesia`, `eleven` |  |
| `credentials` | object | yes |  |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceCredentialsTest`](#schema-voicecredentialstest) |


#### `GET /api/voice-providers/{providerId}/models`

**getAllModels**

- **Auth:** `bearer-jwt`
- **operationId:** `getAllModels`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `providerId` | path | yes | integer (int64) |  |

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoiceProviderModelDto`](#schema-voiceprovidermodeldto)[] |


#### `POST /api/voice-providers/{providerId}/voices`

**getAllVoices**

- **Auth:** `bearer-jwt`
- **operationId:** `getAllVoices`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `providerId` | path | yes | integer (int64) |  |

**Request body** (`application/json`)

```json
{
  "type": "object",
  "additionalProperties": {}
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`Voice`](#schema-voice)[] |


#### `POST /api/voice-providers/{providerId}/voices/{voiceId}/preview`

**getVoicePreview**

- **Auth:** `bearer-jwt`
- **operationId:** `getVoicePreview`

**Parameters**

| Name | In | Required | Type | Description |
|---|---|---|---|---|
| `providerId` | path | yes | integer (int64) |  |
| `voiceId` | path | yes | string |  |

**Request body** (`application/json`)

```json
{
  "type": "object",
  "additionalProperties": {}
}
```

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`VoicePreviewResponse`](#schema-voicepreviewresponse) |


#### ▸ public-health-controller

#### `GET /health`

**health**

- **Auth:** **Public** — no authentication required
- **operationId:** `health`

**Responses**

| Status | Description | Body schema |
|---|---|---|
| `200` | OK | [`HealthResponse`](#schema-healthresponse) |


---

## 12. Schema Reference

All 364 component schemas, alphabetical. Field tables show type, whether required, constraints/enums, and description. Type cells link to nested schemas.

<a id="schema-aiproviderdto"></a>
#### `AIProviderDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string | yes |  |  |
| `availableModels` | [`AiModelDto`](#schema-aimodeldto)[] | yes |  |  |


<a id="schema-addfundsrequest"></a>
#### `AddFundsRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `amount` | integer (int64) | yes |  |  |
| `paymentMethodId` | integer (int64) | yes |  |  |
| `idempotencyKey` | string | yes |  |  |
| `description` | string |  |  |  |


<a id="schema-addfundsresponse"></a>
#### `AddFundsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `transactionId` | integer (int64) | yes |  |  |
| `status` | string | yes | `PENDING`, `COMPLETED`, `FAILED`, `REFUNDED` |  |
| `creditBalanceAfter` | integer (int64) | yes |  |  |
| `stripePaymentIntentId` | string |  |  |  |
| `error` | string |  |  |  |
| `nextAction` | [`NextAction`](#schema-nextaction) |  |  |  |


<a id="schema-addpaymentmethodrequest"></a>
#### `AddPaymentMethodRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `paymentMethodId` | string | yes |  |  |
| `setAsDefault` | boolean | yes |  |  |


<a id="schema-advancedsettings"></a>
#### `AdvancedSettings`

> Advanced settings for agent behavior

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `addNameToContext` | boolean |  |  | Add agent name to context |
| `addDatetimeToContext` | boolean |  |  | Add current datetime to context |
| `addLocationToContext` | boolean |  |  | Add user location to context |


<a id="schema-agentclonerequest"></a>
#### `AgentCloneRequest`

> Request to clone an existing agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned agent |
| `presetName` | string |  | minLength=`1` ; maxLength=`100` | Display name for the cloned agent shown in the platform dashboard |


<a id="schema-agentconfigurationlong"></a>
#### `AgentConfigurationLong`

> Agent configuration settings

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `model` | [`ModelConfigurationLong`](#schema-modelconfigurationlong) |  |  | Model configuration |
| `instructions` | string |  |  | System instructions for the agent |
| `addDatetimeToContext` | boolean |  |  | Add datetime to context (deprecated, use advancedSettings.addDatetimeToContext) |
| `debugMode` | boolean |  |  | Enable debug mode |
| `markdown` | boolean | yes |  | Enable markdown formatting |
| `maxIterations` | integer (int32) |  |  | Maximum iterations |
| `monitoring` | boolean |  |  | Enable monitoring |
| `parallelToolCalls` | boolean |  |  | Enable parallel tool calls |
| `reasoning` | boolean |  |  | Enable reasoning |
| `responseModel` | object |  |  | Response model type (deprecated, use structuredResponseConfig.responseModel) |
| `retryConfig` | [`AiRetryConfig`](#schema-airetryconfig) |  |  | Retry configuration |
| `showToolCalls` | boolean |  |  | Show tool calls in response |
| `structuredOutputs` | boolean |  |  | Enable structured outputs (deprecated, use structuredResponseConfig.structuredOutputs) |
| `toolChoice` | string |  |  | Tool choice strategy |
| `toolExecutionSettings` | [`ToolExecutionSettings`](#schema-toolexecutionsettings) |  |  | Tool execution settings |
| `tools` | [`ToolConfiguration`](#schema-toolconfiguration)[] |  |  | List of tools enabled for this agent |
| `mcp` | [`McpConfiguration`](#schema-mcpconfiguration)[] |  |  | MCP (Model Context Protocol) configurations |
| `httpRequests` | [`HttpRequestReference`](#schema-httprequestreference)[] |  |  | HTTP request references (minimal data, enriched before sending to AI service) |
| `memory` | [`MemoryConfiguration`](#schema-memoryconfiguration) |  |  | Memory configuration |
| `sessionStorage` | [`SessionStorageConfiguration`](#schema-sessionstorageconfiguration) |  |  | Session storage configuration |
| `advancedSettings` | [`AdvancedSettings`](#schema-advancedsettings) |  |  | Advanced settings for agent behavior |
| `structuredResponseConfig` | [`StructuredResponseConfig`](#schema-structuredresponseconfig) |  |  | Structured response configuration |
| `agenticRag` | boolean |  |  | Enable Agentic RAG |
| `systemMetadataFiltering` | boolean |  |  | Enable automatic metadata filtering for knowledge base queries based on system context |
| `knowledgeBase` | [`KnowledgeBaseConfiguration`](#schema-knowledgebaseconfiguration)[] |  |  | Knowledge base configurations with storage resource UUIDs |
| `skills` | [`SkillsConfiguration`](#schema-skillsconfiguration) |  |  | Skills configuration — list of AI service skill UUIDs to load for this agent |
| `metadata` | object |  |  | Metadata including creation method and other tracking information |
| `agentCreator` | boolean | yes |  |  |


<a id="schema-agentconfigurationstring"></a>
#### `AgentConfigurationString`

> Agent configuration settings

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `model` | [`ModelConfigurationString`](#schema-modelconfigurationstring) |  |  | Model configuration |
| `instructions` | string |  |  | System instructions for the agent |
| `addDatetimeToContext` | boolean |  |  | Add datetime to context (deprecated, use advancedSettings.addDatetimeToContext) |
| `debugMode` | boolean |  |  | Enable debug mode |
| `markdown` | boolean | yes |  | Enable markdown formatting |
| `maxIterations` | integer (int32) |  |  | Maximum iterations |
| `monitoring` | boolean |  |  | Enable monitoring |
| `parallelToolCalls` | boolean |  |  | Enable parallel tool calls |
| `reasoning` | boolean |  |  | Enable reasoning |
| `responseModel` | object |  |  | Response model type (deprecated, use structuredResponseConfig.responseModel) |
| `retryConfig` | [`AiRetryConfig`](#schema-airetryconfig) |  |  | Retry configuration |
| `showToolCalls` | boolean |  |  | Show tool calls in response |
| `structuredOutputs` | boolean |  |  | Enable structured outputs (deprecated, use structuredResponseConfig.structuredOutputs) |
| `toolChoice` | string |  |  | Tool choice strategy |
| `toolExecutionSettings` | [`ToolExecutionSettings`](#schema-toolexecutionsettings) |  |  | Tool execution settings |
| `tools` | [`ToolConfiguration`](#schema-toolconfiguration)[] |  |  | List of tools enabled for this agent |
| `mcp` | [`McpConfiguration`](#schema-mcpconfiguration)[] |  |  | MCP (Model Context Protocol) configurations |
| `httpRequests` | [`HttpRequestReference`](#schema-httprequestreference)[] |  |  | HTTP request references (minimal data, enriched before sending to AI service) |
| `memory` | [`MemoryConfiguration`](#schema-memoryconfiguration) |  |  | Memory configuration |
| `sessionStorage` | [`SessionStorageConfiguration`](#schema-sessionstorageconfiguration) |  |  | Session storage configuration |
| `advancedSettings` | [`AdvancedSettings`](#schema-advancedsettings) |  |  | Advanced settings for agent behavior |
| `structuredResponseConfig` | [`StructuredResponseConfig`](#schema-structuredresponseconfig) |  |  | Structured response configuration |
| `agenticRag` | boolean |  |  | Enable Agentic RAG |
| `systemMetadataFiltering` | boolean |  |  | Enable automatic metadata filtering for knowledge base queries based on system context |
| `knowledgeBase` | [`KnowledgeBaseConfiguration`](#schema-knowledgebaseconfiguration)[] |  |  | Knowledge base configurations with storage resource UUIDs |
| `skills` | [`SkillsConfiguration`](#schema-skillsconfiguration) |  |  | Skills configuration — list of AI service skill UUIDs to load for this agent |
| `metadata` | object |  |  | Metadata including creation method and other tracking information |
| `agentCreator` | boolean | yes |  |  |


<a id="schema-agentconfigurationupdaterequest"></a>
#### `AgentConfigurationUpdateRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`1` ; maxLength=`100` |  |
| `presetName` | string |  | minLength=`1` ; maxLength=`100` |  |
| `description` | string |  | minLength=`10` ; maxLength=`1000` |  |
| `configuration` | [`AgentConfigurationString`](#schema-agentconfigurationstring) |  |  |  |


<a id="schema-agentcreaterequest"></a>
#### `AgentCreateRequest`

> Request to create a new agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration |


<a id="schema-agentexecutionresponse"></a>
#### `AgentExecutionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `run_id` | string | yes |  |  |
| `content` | object | yes |  |  |
| `session_id` | string | yes |  |  |
| `user_id` | string | yes |  |  |
| `messages` | object[] |  |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-agentjobcreateddto"></a>
#### `AgentJobCreatedDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `jobId` | string | yes |  |  |
| `agentId` | string | yes |  |  |
| `status` | string | yes | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `TIMEOUT`, `CANCELLED` |  |
| `userId` | string | yes |  |  |
| `sessionId` | string | yes |  |  |
| `message` | string | yes |  |  |


<a id="schema-agentjobresponsedto"></a>
#### `AgentJobResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `jobId` | string | yes |  |  |
| `agentId` | string | yes |  |  |
| `status` | string | yes | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `TIMEOUT`, `CANCELLED` |  |
| `userId` | string |  |  |  |
| `sessionId` | string |  |  |  |
| `result` | [`AgentJobResult`](#schema-agentjobresult) |  |  |  |
| `error` | string |  |  |  |
| `startedAt` | string (date-time) |  |  |  |
| `completedAt` | string (date-time) |  |  |  |


<a id="schema-agentjobresult"></a>
#### `AgentJobResult`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `runId` | string | yes |  |  |
| `content` | object | yes |  |  |
| `sessionId` | string | yes |  |  |
| `userId` | string | yes |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `messages` | object[] |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-agentoptionv1dto"></a>
#### `AgentOptionV1Dto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `aiServiceAgentId` | string | yes |  |  |


<a id="schema-agentresponse"></a>
#### `AgentResponse`

> Agent information

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Internal database ID |
| `aiServiceAgentId` | string |  |  | AI service agent ID (UUID) |
| `name` | string | yes |  | LLM-facing agent name known to the AI |
| `presetName` | string | yes |  | Display name shown in the platform dashboard |
| `description` | string |  |  | Agent description |
| `organizationId` | integer (int64) | yes |  | Internal numeric organization ID. Deprecated — use organizationRefId. |
| `organizationRefId` | string | yes |  | Organization reference ID |
| `projectId` | integer (int64) | yes |  | Internal numeric project ID. Deprecated — use projectRefId. |
| `projectRefId` | string | yes |  | Project reference ID |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration |
| `created` | string (date-time) | yes |  | Creation timestamp |
| `createdBy` | string | yes |  | Created by user |
| `updated` | string (date-time) | yes |  | Last update timestamp |
| `updatedBy` | string | yes |  | Updated by user |
| `status` | string |  |  | Agent status from AI service |
| `supportAttachments` | boolean | yes |  | Whether agent supports file attachments |
| `supportedAttachmentTypes` | string[] | yes |  | List of supported attachment MIME types |


<a id="schema-agentrundto"></a>
#### `AgentRunDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `runId` | string |  |  |  |
| `agentName` | string |  |  |  |
| `model` | string |  |  |  |
| `status` | string | yes | `SUCCESS`, `FAILED` |  |
| `created` | string (date-time) | yes |  |  |
| `duration` | number |  |  |  |
| `inputTokens` | integer (int32) | yes |  |  |
| `outputTokens` | integer (int32) | yes |  |  |
| `userName` | string |  |  |  |
| `apiKeyName` | string |  |  |  |
| `projectName` | string | yes |  |  |
| `creditsUsed` | integer (int32) | yes |  |  |
| `voiceDurationS` | number | yes |  |  |
| `ttsChars` | integer (int32) | yes |  |  |
| `sttDurationS` | number | yes |  |  |
| `documentPageCount` | integer (int32) |  |  |  |


<a id="schema-agentrunresponsedata"></a>
#### `AgentRunResponseData`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `run_id` | string | yes |  |  |
| `content` | object | yes |  |  |
| `session_id` | string | yes |  |  |
| `user_id` | string | yes |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `messages` | object[] |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-agentupdaterequest"></a>
#### `AgentUpdateRequest`

> Request to update an existing agent

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration |


<a id="schema-agentusagedtoobject"></a>
#### `AgentUsageDtoObject`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | object | yes |  |  |
| `name` | string | yes |  |  |
| `scope` | string | yes | `TEAM`, `AGENT`, `WORKFLOW` |  |
| `teamId` | object |  |  |  |
| `referenceId` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `avgSuccessRate` | number | yes |  |  |
| `avgExecutionDuration` | number | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalInputTokens` | integer (int64) | yes |  |  |
| `totalOutputTokens` | integer (int64) | yes |  |  |
| `totalToolCalls` | integer (int64) | yes |  |  |
| `totalRagRetrievals` | integer (int64) | yes |  |  |
| `totalTtsChars` | integer (int64) | yes |  |  |
| `totalSttDuration` | number | yes |  |  |
| `totalAudioDuration` | number | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |
| `usage` | [`UsageDto`](#schema-usagedto)[] | yes |  |  |


<a id="schema-agentv1createrequest"></a>
#### `AgentV1CreateRequest`

> Request to create a new agent via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `projectRefId` | string |  |  | Reference ID of the project this agent belongs to (e.g. 'proj_...'). Use this instead of projectId. |
| `projectId` | integer (int64) |  |  | Numeric ID of the project. Deprecated — use projectRefId. |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |


<a id="schema-agentv1updaterequest"></a>
#### `AgentV1UpdateRequest`

> Request to update an existing agent via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing agent name known to the AI (e.g. 'Clara') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Agent') |
| `description` | string |  |  | Agent description |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  | Agent configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |


<a id="schema-agentversioncomparisonresponse"></a>
#### `AgentVersionComparisonResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `version1` | [`AgentVersionResponse`](#schema-agentversionresponse) | yes |  |  |
| `version2` | [`AgentVersionResponse`](#schema-agentversionresponse) | yes |  |  |
| `differences` | [`ConfigurationDifference`](#schema-configurationdifference)[] | yes |  |  |


<a id="schema-agentversionresponse"></a>
#### `AgentVersionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `versionNumber` | integer (int32) | yes |  |  |
| `name` | string | yes |  |  |
| `presetName` | string | yes |  |  |
| `description` | string |  |  |  |
| `configuration` | [`AgentConfigurationLong`](#schema-agentconfigurationlong) | yes |  |  |
| `createdAt` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `changeSummary` | string |  |  |  |
| `rollbackSourceVersionId` | integer (int64) |  |  |  |
| `isCurrentVersion` | boolean | yes |  |  |


<a id="schema-aimodeldto"></a>
#### `AiModelDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `description` | string | yes |  |  |
| `contextLength` | integer (int32) | yes |  |  |
| `maxOutputTokens` | integer (int32) | yes |  |  |
| `fixedTemperature` | integer (int32) |  |  |  |


<a id="schema-airetryconfig"></a>
#### `AiRetryConfig`

> Retry configuration for AI service

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `retries` | integer (int32) |  |  | Number of retry attempts |
| `delayBetweenRetries` | integer (int32) |  |  | Delay in seconds between retries |
| `exponentialBackoff` | boolean |  |  | Whether to use exponential backoff |


<a id="schema-aitoolinfo"></a>
#### `AiToolInfo`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `description` | string |  |  |  |
| `category` | string |  |  |  |
| `requiredParams` | object |  |  |  |
| `optionalParams` | object |  |  |  |
| `configSchema` | object |  |  |  |
| `availableMethods` | [`AiToolMethod`](#schema-aitoolmethod)[] |  |  |  |


<a id="schema-aitoolmethod"></a>
#### `AiToolMethod`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `description` | string |  |  |  |
| `parameters` | object |  |  |  |


<a id="schema-aiworkflowdetailsresponse"></a>
#### `AiWorkflowDetailsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string |  |  |  |
| `inputSchema` | [`JsonSchema`](#schema-jsonschema) |  |  |  |


<a id="schema-aiworkflowlistitem"></a>
#### `AiWorkflowListItem`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string |  |  |  |


<a id="schema-aiworkflowrunresponse"></a>
#### `AiWorkflowRunResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `runId` | string |  |  |  |
| `workflowId` | string |  |  |  |
| `workflowName` | string |  |  |  |
| `sessionId` | string |  |  |  |
| `userId` | string |  |  |  |
| `input` | object |  |  |  |
| `content` | object |  |  |  |
| `contentType` | string |  |  |  |
| `result` | object |  |  |  |
| `messages` | object[] |  |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `status` | string |  |  |  |
| `createdAt` | integer (int64) |  |  |  |


<a id="schema-alertconfig"></a>
#### `AlertConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `message` | string | yes | minLength=`1` |  |
| `channels` | string[] | yes | `EMAIL`, `WEBHOOK` |  |
| `users` | string[] | yes |  |  |


<a id="schema-apikeydto"></a>
#### `ApiKeyDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) |  |  |  |
| `referenceId` | string |  |  |  |
| `key` | string |  |  |  |
| `name` | string |  |  |  |
| `created` | string (date-time) |  |  |  |
| `updated` | string (date-time) |  |  |  |
| `lastUsed` | string (date-time) |  |  |  |
| `isActive` | boolean | yes |  |  |
| `projects` | [`ApiKeyProjectDto`](#schema-apikeyprojectdto)[] |  |  |  |
| `createdBy` | [`BaseUserDto`](#schema-baseuserdto) |  |  |  |
| `active` | boolean |  |  |  |


<a id="schema-apikeyoptiondto"></a>
#### `ApiKeyOptionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-apikeyprojectdto"></a>
#### `ApiKeyProjectDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `referenceId` | string | yes |  |  |


<a id="schema-apikeyrequest"></a>
#### `ApiKeyRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `projectIds` | integer (int64)[] | yes |  |  |


<a id="schema-apikeysetupresponsedto"></a>
#### `ApiKeySetupResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKey` | [`ApiKeyDto`](#schema-apikeydto) | yes |  |  |
| `project` | [`ProjectDto`](#schema-projectdto) | yes |  |  |


<a id="schema-apikeyv1request"></a>
#### `ApiKeyV1Request`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |


<a id="schema-apiresponseagentresponse"></a>
#### `ApiResponseAgentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AgentResponse`](#schema-agentresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseagentrunresponsedata"></a>
#### `ApiResponseAgentRunResponseData`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AgentRunResponseData`](#schema-agentrunresponsedata) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseaiworkflowdetailsresponse"></a>
#### `ApiResponseAiWorkflowDetailsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AiWorkflowDetailsResponse`](#schema-aiworkflowdetailsresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseapikeydto"></a>
#### `ApiResponseApiKeyDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`ApiKeyDto`](#schema-apikeydto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsecrawlresultdto"></a>
#### `ApiResponseCrawlResultDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`CrawlResultDto`](#schema-crawlresultdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseevalrundetaildto"></a>
#### `ApiResponseEvalRunDetailDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`EvalRunDetailDto`](#schema-evalrundetaildto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsefilelistdto"></a>
#### `ApiResponseFileListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`FileListDto`](#schema-filelistdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsefileuploadresultdto"></a>
#### `ApiResponseFileUploadResultDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`FileUploadResultDto`](#schema-fileuploadresultdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsehealthstatus"></a>
#### `ApiResponseHealthStatus`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`HealthStatus`](#schema-healthstatus) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsehttprequestresponse"></a>
#### `ApiResponseHttpRequestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`HttpRequestResponse`](#schema-httprequestresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsehttprequesttestresponse"></a>
#### `ApiResponseHttpRequestTestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`HttpRequestTestResponse`](#schema-httprequesttestresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistagentresponse"></a>
#### `ApiResponseListAgentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AgentResponse`](#schema-agentresponse)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistaiworkflowlistitem"></a>
#### `ApiResponseListAiWorkflowListItem`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AiWorkflowListItem`](#schema-aiworkflowlistitem)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselisthttprequestresponse"></a>
#### `ApiResponseListHttpRequestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`HttpRequestResponse`](#schema-httprequestresponse)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistllmproviderconfigwithmodelsresponse"></a>
#### `ApiResponseListLlmProviderConfigWithModelsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`LlmProviderConfigWithModelsResponse`](#schema-llmproviderconfigwithmodelsresponse)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistprojectdto"></a>
#### `ApiResponseListProjectDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`ProjectDto`](#schema-projectdto)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistsessionrundto"></a>
#### `ApiResponseListSessionRunDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SessionRunDto`](#schema-sessionrundto)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistsimpletagdto"></a>
#### `ApiResponseListSimpleTagDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SimpleTagDto`](#schema-simpletagdto)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseliststorageresourceresponse"></a>
#### `ApiResponseListStorageResourceResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`StorageResourceResponse`](#schema-storageresourceresponse)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistteamresponse"></a>
#### `ApiResponseListTeamResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TeamResponse`](#schema-teamresponse)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponselistvoicepresetclientdto"></a>
#### `ApiResponseListVoicePresetClientDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`VoicePresetClientDto`](#schema-voicepresetclientdto)[] |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsemapstringobject"></a>
#### `ApiResponseMapStringObject`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | object |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseorganizationlimitsdto"></a>
#### `ApiResponseOrganizationLimitsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`OrganizationLimitsDto`](#schema-organizationlimitsdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseoutboundcallresponsedto"></a>
#### `ApiResponseOutboundCallResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`OutboundCallResponseDto`](#schema-outboundcallresponsedto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseprojectdto"></a>
#### `ApiResponseProjectDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`ProjectDto`](#schema-projectdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsesessiondto"></a>
#### `ApiResponseSessionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SessionDto`](#schema-sessiondto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseskillexecuteresponse"></a>
#### `ApiResponseSkillExecuteResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SkillExecuteResponse`](#schema-skillexecuteresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseskillresponse"></a>
#### `ApiResponseSkillResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SkillResponse`](#schema-skillresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsestorageresourceresponse"></a>
#### `ApiResponseStorageResourceResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`StorageResourceResponse`](#schema-storageresourceresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsestring"></a>
#### `ApiResponseString`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | string |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseteamresponse"></a>
#### `ApiResponseTeamResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TeamResponse`](#schema-teamresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseteamrunresponsedata"></a>
#### `ApiResponseTeamRunResponseData`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TeamRunResponseData`](#schema-teamrunresponsedata) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsetelephonypresetresponsedto"></a>
#### `ApiResponseTelephonyPresetResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsetracedetaildto"></a>
#### `ApiResponseTraceDetailDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TraceDetailDto`](#schema-tracedetaildto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseunit"></a>
#### `ApiResponseUnit`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponseusageresponsedto"></a>
#### `ApiResponseUsageResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`UsageResponseDto`](#schema-usageresponsedto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsevoicepresetdto"></a>
#### `ApiResponseVoicePresetDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`VoicePresetDto`](#schema-voicepresetdto) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-apiresponsevoicesessionresponse"></a>
#### `ApiResponseVoiceSessionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`VoiceSessionResponse`](#schema-voicesessionresponse) |  |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-appconfigdto"></a>
#### `AppConfigDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `baseUrl` | string | yes |  |  |
| `deploymentMode` | string | yes | `SAAS`, `ENTERPRISE` |  |
| `googleAuthEnabled` | boolean | yes |  |  |


<a id="schema-audiosessionnotification"></a>
#### `AudioSessionNotification`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `agentId` | string | yes |  |  |
| `sessionId` | string | yes |  |  |
| `participantIdentity` | string | yes |  |  |
| `type` | string | yes | `audio_session_started`, `audio_session_ended` |  |
| `stt_time_s` | number (double) | yes |  |  |
| `tts_char_count` | integer (int32) | yes |  |  |
| `mic_sample_rate` | integer (int32) |  |  |  |
| `mic_channels` | integer (int32) |  |  |  |
| `tts_sample_rate` | integer (int32) |  |  |  |
| `tts_voice_provider` | string | yes |  |  |
| `tts_voice_model_id` | string | yes |  |  |
| `stt_default_language` | string | yes |  |  |
| `total_call_duration_s` | number (double) | yes |  |  |


<a id="schema-autorechargeconfigrequest"></a>
#### `AutoRechargeConfigRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `enabled` | boolean | yes |  |  |
| `threshold` | integer (int64) | yes |  |  |
| `amount` | integer (int64) | yes |  |  |
| `monthlyLimit` | integer (int64) | yes |  |  |


<a id="schema-autorechargeconfigresponse"></a>
#### `AutoRechargeConfigResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `enabled` | boolean | yes |  |  |
| `threshold` | integer (int64) | yes |  |  |
| `amount` | integer (int64) | yes |  |  |
| `monthlyLimit` | integer (int64) | yes |  |  |


<a id="schema-baseagentoptiondto"></a>
#### `BaseAgentOptionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `refId` | string | yes |  |  |
| `name` | string | yes |  |  |
| `presetName` | string | yes |  |  |
| `isDeleted` | boolean | yes |  |  |


<a id="schema-baseidnamedto"></a>
#### `BaseIdNameDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-baseorganizationmemberdto"></a>
#### `BaseOrganizationMemberDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) | yes |  |  |


<a id="schema-baseprojectdto"></a>
#### `BaseProjectDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |


<a id="schema-baseprojectmemberdto"></a>
#### `BaseProjectMemberDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |


<a id="schema-baseuserdto"></a>
#### `BaseUserDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `email` | string | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |


<a id="schema-baseuseroptiondto"></a>
#### `BaseUserOptionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `refId` | string | yes |  |  |
| `email` | string | yes |  |  |
| `lastName` | string | yes |  |  |
| `firstName` | string | yes |  |  |


<a id="schema-blockconfig"></a>
#### `BlockConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `message` | string | yes | minLength=`1` |  |


<a id="schema-canceldeploymentresponse"></a>
#### `CancelDeploymentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `deploymentId` | integer (int64) | yes |  |  |
| `message` | string | yes |  |  |


<a id="schema-chunkingstrategiesresponse"></a>
#### `ChunkingStrategiesResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `strategies` | object | yes |  |  |
| `timestamp` | string (date-time) | yes |  |  |


<a id="schema-chunkingstrategy"></a>
#### `ChunkingStrategy`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string | yes |  |  |
| `parameters` | object | yes |  |  |


<a id="schema-complianceframeworkdto"></a>
#### `ComplianceFrameworkDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string |  |  |  |


<a id="schema-configurationdifference"></a>
#### `ConfigurationDifference`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `path` | string | yes |  |  |
| `operation` | string | yes | `ADDED`, `REMOVED`, `MODIFIED` |  |
| `oldValue` | object |  |  |  |
| `newValue` | object |  |  |  |


<a id="schema-continueagentrunrequest"></a>
#### `ContinueAgentRunRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionId` | string | yes |  | Session ID for conversation continuity |
| `tools` | string |  |  |  |


<a id="schema-crawlrequest"></a>
#### `CrawlRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `urls` | [`CrawlUrlRequest`](#schema-crawlurlrequest)[] | yes |  |  |
| `ignoreLinks` | boolean | yes |  |  |


<a id="schema-crawlresultdto"></a>
#### `CrawlResultDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `message` | string | yes |  |  |
| `vectorStoreId` | string | yes |  |  |
| `urlsCount` | integer (int32) | yes |  |  |
| `taskIds` | string[] | yes |  |  |
| `timestamp` | string (date-time) | yes |  |  |


<a id="schema-crawlurlrequest"></a>
#### `CrawlUrlRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `url` | string | yes | minLength=`1` ; pattern=`^https?://.*` |  |
| `isRoot` | boolean | yes |  |  |


<a id="schema-createagentdirectrequest"></a>
#### `CreateAgentDirectRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `initialMessage` | string | yes | minLength=`10` ; maxLength=`1000` |  |


<a id="schema-createoutboundcalldto"></a>
#### `CreateOutboundCallDto`

> Request to initiate an outbound call

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `fromNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Your outbound phone number to call from (must be in telephony preset) |
| `toNumber` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Phone number to call |
| `playDialtone` | boolean | yes |  | Play dial tone to room until call is answered |


<a id="schema-createsecretrequest"></a>
#### `CreateSecretRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string | yes | minLength=`0` ; maxLength=`255` |  |
| `value` | string | yes | minLength=`1` |  |
| `description` | string |  | minLength=`0` ; maxLength=`1000` |  |


<a id="schema-createtelephonypresetdto"></a>
#### `CreateTelephonyPresetDto`

> Request DTO for creating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |


<a id="schema-createwebhooksubscriptionrequest"></a>
#### `CreateWebhookSubscriptionRequest`

> Request to create a webhook subscription

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`255` | Name of the webhook subscription |
| `endpointUrl` | string | yes | minLength=`0` ; maxLength=`2048` ; pattern=`^https?://.*` | The webhook endpoint URL |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Description of the webhook subscription |
| `eventTypes` | string[] | yes |  | List of event types to subscribe to |
| `maxDeliveryAttempts` | integer (int32) | yes | minimum=`1` ; maximum=`10` | Maximum number of delivery attempts |


<a id="schema-createwithassistantrequest"></a>
#### `CreateWithAssistantRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string |  | minLength=`10` ; maxLength=`1000` |  |
| `provider` | integer (int64) | yes |  |  |
| `modelName` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `creatorOnly` | boolean | yes |  |  |


<a id="schema-creditdiscounttierdto"></a>
#### `CreditDiscountTierDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `minAmount` | integer (int64) | yes |  |  |
| `discountPercent` | integer (int32) | yes |  |  |


<a id="schema-customheader"></a>
#### `CustomHeader`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `header` | string | yes |  |  |
| `value` | string | yes |  |  |


<a id="schema-customerbillingdto"></a>
#### `CustomerBillingDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `userId` | integer (int64) | yes |  |  |
| `stripeCustomerId` | string | yes |  |  |
| `currency` | string | yes | `USD`, `EUR` |  |
| `autoRecharge` | boolean | yes |  |  |
| `autoRechargeThreshold` | integer (int64) | yes |  |  |
| `autoRechargeMonthlyLimit` | integer (int64) |  |  |  |
| `autoRechargeAmount` | integer (int64) |  |  |  |
| `metadata` | object | yes |  |  |
| `creditBalance` | integer (int64) | yes |  |  |
| `creditRate` | number | yes |  |  |
| `discountTiers` | [`CreditDiscountTierDto`](#schema-creditdiscounttierdto)[] | yes |  |  |


<a id="schema-customerbillinginfo"></a>
#### `CustomerBillingInfo`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `companyName` | string |  |  |  |
| `billingEmail` | string |  |  |  |
| `country` | string |  |  |  |
| `address1` | string |  |  |  |
| `address2` | string |  |  |  |
| `city` | string |  |  |  |
| `state` | string |  |  |  |
| `zip` | string |  |  |  |
| `vat` | string |  |  |  |


<a id="schema-deletedfilesresponse"></a>
#### `DeletedFilesResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `deletedFilesCount` | integer (int32) | yes |  |  |


<a id="schema-deploymentlogdto"></a>
#### `DeploymentLogDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `logLevel` | string | yes |  |  |
| `stage` | string |  |  |  |
| `message` | string | yes |  |  |
| `createdAt` | string (date-time) | yes |  |  |


<a id="schema-deploymentresponse"></a>
#### `DeploymentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `deploymentId` | integer (int64) | yes |  |  |
| `status` | string | yes |  |  |
| `subdomain` | string |  |  |  |
| `accessUrl` | string | yes |  |  |
| `sshUsername` | string | yes |  |  |
| `n8nUsername` | string | yes |  |  |
| `workersCount` | integer (int32) | yes |  |  |
| `createdAt` | string (date-time) | yes |  |  |


<a id="schema-deploymentstatusresponse"></a>
#### `DeploymentStatusResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `deploymentId` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `status` | string | yes |  |  |
| `subdomain` | string |  |  |  |
| `customUrl` | string |  |  |  |
| `accessUrl` | string | yes |  |  |
| `serverIp` | string | yes |  |  |
| `sshUsername` | string | yes |  |  |
| `n8nUsername` | string | yes |  |  |
| `port` | integer (int32) | yes |  |  |
| `deploymentWorkerId` | string |  |  |  |
| `errorMessage` | string |  |  |  |
| `createdAt` | string (date-time) | yes |  |  |
| `updatedAt` | string (date-time) | yes |  |  |
| `completedAt` | string (date-time) |  |  |  |


<a id="schema-documentationsectiondto"></a>
#### `DocumentationSectionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `slug` | string | yes |  |  |
| `description` | string |  |  |  |
| `displayOrder` | integer (int32) | yes |  |  |
| `version` | integer (int32) | yes |  |  |
| `organizationId` | integer (int64) |  |  |  |
| `subsections` | [`DocumentationSubsectionSummaryDto`](#schema-documentationsubsectionsummarydto)[] | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-documentationsectionwithfullsubsectionsdto"></a>
#### `DocumentationSectionWithFullSubsectionsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `slug` | string | yes |  |  |
| `description` | string |  |  |  |
| `displayOrder` | integer (int32) | yes |  |  |
| `version` | integer (int32) | yes |  |  |
| `organizationId` | integer (int64) |  |  |  |
| `subsections` | [`DocumentationSubsectionDto`](#schema-documentationsubsectiondto)[] | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-documentationsubsectiondto"></a>
#### `DocumentationSubsectionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `title` | string | yes |  |  |
| `slug` | string | yes |  |  |
| `content` | string |  |  |  |
| `contentType` | string | yes |  |  |
| `displayOrder` | integer (int32) | yes |  |  |
| `version` | integer (int32) | yes |  |  |
| `sectionId` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-documentationsubsectionsummarydto"></a>
#### `DocumentationSubsectionSummaryDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `title` | string | yes |  |  |
| `slug` | string | yes |  |  |


<a id="schema-documentationupdateresponse"></a>
#### `DocumentationUpdateResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `version` | integer (int32) | yes |  |  |
| `lastModified` | string (date-time) | yes |  |  |


<a id="schema-embeddingsrequest"></a>
#### `EmbeddingsRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `model` | string | yes |  |  |
| `input` | string | yes |  |  |
| `encoding_format` | string |  |  |  |
| `dimensions` | integer (int32) |  |  |  |


<a id="schema-evalrundetaildto"></a>
#### `EvalRunDetailDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string |  |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `workflow_id` | string |  |  |  |
| `model_id` | string |  |  |  |
| `model_provider` | string |  |  |  |
| `name` | string |  |  |  |
| `eval_type` | string |  |  |  |
| `eval_data` | object |  |  |  |
| `eval_input` | object |  |  |  |
| `created_at` | string |  |  |  |
| `updated_at` | string |  |  |  |


<a id="schema-evalruninputrequest"></a>
#### `EvalRunInputRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `eval_type` | string | yes |  |  |
| `input` | string | yes |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `model_id` | string |  |  |  |
| `model_provider` | string |  |  |  |
| `provider_ref_id` | string |  |  |  |
| `name` | string |  |  |  |
| `expected_output` | string |  |  |  |
| `criteria` | string |  |  |  |
| `scoring_strategy` | string |  |  |  |
| `threshold` | integer (int32) |  |  |  |
| `num_iterations` | integer (int32) | yes |  |  |
| `warmup_runs` | integer (int32) | yes |  |  |
| `additional_guidelines` | string |  |  |  |
| `additional_context` | string |  |  |  |
| `expected_tool_calls` | string[] |  |  |  |


<a id="schema-evalrunlistdto"></a>
#### `EvalRunListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `workflow_id` | string |  |  |  |
| `model_id` | string |  |  |  |
| `model_provider` | string |  |  |  |
| `name` | string | yes |  |  |
| `eval_type` | string | yes |  |  |
| `created_at` | string | yes |  |  |
| `updated_at` | string | yes |  |  |


<a id="schema-failovermodelconfiglong"></a>
#### `FailoverModelConfigLong`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | integer (int64) | yes |  | Provider identifier (Long for database, String for AI service) |
| `modelName` | string | yes |  | Model name |
| `temperature` | number (double) |  | minimum=`0` ; maximum=`2` |  |
| `maxTokens` | integer (int32) |  |  |  |
| `providerRefId` | string |  |  |  |
| `providerName` | string |  |  |  |
| `reasoningConfig` | object |  |  |  |


<a id="schema-failovermodelconfigstring"></a>
#### `FailoverModelConfigString`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  | Provider identifier (Long for database, String for AI service) |
| `modelName` | string | yes |  | Model name |
| `temperature` | number (double) |  | minimum=`0` ; maximum=`2` |  |
| `maxTokens` | integer (int32) |  |  |  |
| `providerRefId` | string |  |  |  |
| `providerName` | string |  |  |  |
| `reasoningConfig` | object |  |  |  |


<a id="schema-filedetaildto"></a>
#### `FileDetailDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `fileName` | string | yes |  |  |
| `fileSize` | integer (int64) | yes |  |  |
| `fileType` | string | yes |  |  |
| `status` | string | yes |  |  |
| `error` | string |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `errorSource` | string |  |  |  |
| `type` | string | yes |  |  |
| `chunkingStrategy` | string |  |  |  |
| `chunkingConfig` | object |  |  |  |
| `metadata` | object |  |  |  |
| `filterMetadata` | object |  |  |  |


<a id="schema-filedto"></a>
#### `FileDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `type` | string | yes |  |  |
| `file` | [`FileDetailDto`](#schema-filedetaildto) |  |  |  |
| `rootUrl` | [`RootUrlDto`](#schema-rooturldto) |  |  |  |
| `singlePageUrl` | [`SinglePageUrlDto`](#schema-singlepageurldto) |  |  |  |


<a id="schema-filelistdto"></a>
#### `FileListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `items` | [`FileDto`](#schema-filedto)[] | yes |  |  |
| `total` | integer (int32) | yes |  |  |
| `page` | integer (int32) | yes |  |  |
| `size` | integer (int32) | yes |  |  |
| `pages` | integer (int32) | yes |  |  |


<a id="schema-fileuploadlimitsdto"></a>
#### `FileUploadLimitsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `maxFileSizeMb` | integer (int32) | yes |  |  |
| `maxTotalRequestSizeMb` | integer (int32) | yes |  |  |
| `maxFilesPerRequest` | integer (int32) | yes |  |  |
| `fileTypeSizeLimitsMb` | object | yes |  |  |


<a id="schema-fileuploadresultdto"></a>
#### `FileUploadResultDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `message` | string | yes |  |  |
| `totalFiles` | integer (int32) | yes |  |  |
| `validationErrors` | string[] |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |


<a id="schema-forgotpasswordrequest"></a>
#### `ForgotPasswordRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `email` | string | yes |  |  |


<a id="schema-googlecallbackrequest"></a>
#### `GoogleCallbackRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `code` | string | yes |  |  |
| `redirectUri` | string | yes |  |  |


<a id="schema-healthresponse"></a>
#### `HealthResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `status` | string | yes |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `service` | string | yes |  |  |


<a id="schema-healthstatus"></a>
#### `HealthStatus`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `status` | string | yes |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `apiVersion` | string | yes |  |  |
| `service` | string | yes |  |  |


<a id="schema-httprequestcreaterequest"></a>
#### `HttpRequestCreateRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object | yes |  |  |
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object | yes |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean | yes |  |  |


<a id="schema-httprequestparametermapping"></a>
#### `HttpRequestParameterMapping`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `enabled` | boolean | yes |  |  |
| `metadataKey` | string | yes |  |  |
| `toolParameter` | string | yes |  |  |
| `transformationType` | string | yes |  |  |
| `transformationConfig` | string |  |  |  |


<a id="schema-httprequestreference"></a>
#### `HttpRequestReference`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `httpRequestId` | string | yes |  | HTTP request ID (UUID) |
| `httpRequestName` | string | yes |  | HTTP request name |
| `enabled` | boolean | yes |  | Whether this HTTP request is enabled |


<a id="schema-httprequestresponse"></a>
#### `HttpRequestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `projectId` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string |  |  |  |
| `headers` | object | yes |  |  |
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes |  |  |
| `inputJsonSchema` | object | yes |  |  |
| `outputJsonSchema` | object |  |  |  |
| `requestId` | string (uuid) | yes |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `updatedBy` | string | yes |  |  |
| `deletedAt` | string (date-time) |  |  |  |


<a id="schema-httprequesttestdata"></a>
#### `HttpRequestTestData`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `httpMethod` | string | yes | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string | yes |  |  |
| `headers` | object | yes |  |  |
| `data` | object |  |  |  |


<a id="schema-httprequesttestresponse"></a>
#### `HttpRequestTestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `status` | integer (int32) |  |  |  |
| `error` | string |  |  |  |
| `data` | object |  |  |  |
| `headers` | object |  |  |  |


<a id="schema-httprequestupdaterequest"></a>
#### `HttpRequestUpdateRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `headers` | object |  |  |  |
| `httpMethod` | string |  | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS` |  |
| `url` | string |  | minLength=`0` ; maxLength=`2048` |  |
| `inputJsonSchema` | object |  |  |  |
| `outputJsonSchema` | object |  |  |  |
| `parameterMappings` | [`HttpRequestParameterMapping`](#schema-httprequestparametermapping)[] | yes |  |  |
| `passMetadataToHeaders` | boolean |  |  |  |


<a id="schema-importconfigurationsresponse"></a>
#### `ImportConfigurationsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `imported` | integer (int32) | yes |  |  |
| `skipped` | integer (int32) | yes |  |  |


<a id="schema-inboundconfigdto"></a>
#### `InboundConfigDto`

> Inbound telephony configuration

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `numbers` | [`InboundPhoneConfigDto`](#schema-inboundphoneconfigdto)[] |  |  | List of inbound phone numbers with voice presets |
| `ips` | string[] | yes |  | List of allowed IP addresses for inbound connections |
| `allowedNumbers` | string[] | yes |  | Restrict incoming calls to only these phone numbers (E.164 format). Leave empty to allow all. |
| `mediaEncryption` | string | yes | `SIP_MEDIA_ENCRYPT_DISABLE`, `SIP_MEDIA_ENCRYPT_ALLOW`, `SIP_MEDIA_ENCRYPT_REQUIRE` | Media encryption mode |
| `includeHeaders` | string | yes | `SIP_NO_HEADERS`, `SIP_X_HEADERS`, `SIP_ALL_HEADERS` | SIP headers inclusion mode |
| `enableKrisp` | boolean | yes |  | Enable Krisp noise cancellation |


<a id="schema-inboundphoneconfigdto"></a>
#### `InboundPhoneConfigDto`

> Inbound phone number configuration with voice preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `number` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Phone number (digits only, optional + prefix) |
| `voicePresetId` | string | yes | minLength=`1` | Voice preset ID to use for this number |


<a id="schema-inboundsessionrequest"></a>
#### `InboundSessionRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `participantIdentity` | string | yes | minLength=`1` | Participant identity |
| `voicePresetReferenceId` | string | yes | minLength=`1` | Reference ID of the voice preset |


<a id="schema-inboundsessionresponse"></a>
#### `InboundSessionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionId` | string | yes |  | Generated session ID |
| `policies` | [`PolicyItemForEvaluation`](#schema-policyitemforevaluation)[] | yes |  | Effective project-level policies for this session |


<a id="schema-initiatedeploymentrequest"></a>
#### `InitiateDeploymentRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `subdomain` | string |  | minLength=`0` ; maxLength=`63` ; pattern=`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` |  |
| `serverIp` | string | yes | minLength=`1` ; pattern=`^((25[0-5]|(2[0-4]|1\d|[1-9]|)\d)\.?\b){4}$` |  |
| `sshUsername` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `n8nUsername` | string (email) | yes | minLength=`1` |  |
| `n8nPassword` | string | yes | minLength=`12` ; maxLength=`2147483647` |  |
| `port` | integer (int32) | yes | minimum=`1` ; maximum=`65535` |  |
| `workersCount` | integer (int32) | yes | minimum=`1` ; maximum=`4` |  |
| `customUrl` | string |  | minLength=`0` ; maxLength=`500` |  |


<a id="schema-invitationdetails"></a>
#### `InvitationDetails`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `email` | string | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |
| `organizationName` | string |  |  |  |
| `projectName` | string |  |  |  |
| `hasAccount` | boolean |  |  |  |


<a id="schema-inviteorgmembersdto"></a>
#### `InviteOrgMembersDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `emails` | string[] | yes |  |  |
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) | yes |  |  |


<a id="schema-inviteprojectmemberdto"></a>
#### `InviteProjectMemberDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `userIds` | integer (int64)[] |  |  |  |
| `emails` | string[] |  |  |  |
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |


<a id="schema-jsonschema"></a>
#### `JsonSchema`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `type` | string |  |  |  |
| `title` | string |  |  |  |
| `description` | string |  |  |  |
| `properties` | object |  |  |  |
| `required` | string[] |  |  |  |


<a id="schema-jsonschemaproperty"></a>
#### `JsonSchemaProperty`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `type` | string |  |  |  |
| `title` | string |  |  |  |
| `description` | string |  |  |  |
| `default` | object |  |  |  |
| `anyOf` | object[] |  |  |  |
| `format` | string |  |  |  |
| `items` | object |  |  |  |
| `additionalProperties` | object |  |  |  |
| `examples` | object[] |  |  |  |
| `primaryType` | string |  |  |  |
| `isNullable` | boolean | yes |  |  |


<a id="schema-keyusagedto"></a>
#### `KeyUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `referenceId` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `totalInputTokens` | integer (int64) | yes |  |  |
| `totalOutputTokens` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `avgSuccessRate` | number | yes |  |  |
| `avgExecutionDuration` | number | yes |  |  |
| `totalToolCalls` | integer (int64) | yes |  |  |
| `totalRagRetrievals` | integer (int64) | yes |  |  |
| `totalAudioDuration` | number | yes |  |  |
| `totalTtsChars` | integer (int64) | yes |  |  |
| `totalSttDuration` | number | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |
| `usage` | [`UsageDto`](#schema-usagedto)[] | yes |  |  |


<a id="schema-knowledgebaseconfiguration"></a>
#### `KnowledgeBaseConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  |  | Knowledge base name |
| `description` | string |  |  | Knowledge base description |
| `type` | string |  |  | Knowledge base type |
| `knowledgeBaseId` | string | yes |  | Knowledge base ID - references storage_resource.vector_store_id |
| `searchType` | string |  |  | Search type |
| `hybridSearchAlpha` | number (double) |  |  | Hybrid search alpha (0.0-1.0). Controls the balance between keyword and vector search. 0.0 = pure keyword, 1.0 = pure vector. |
| `maxResults` | integer (int32) |  |  | Maximum number of results to return |
| `addReferences` | boolean |  |  | Whether to add references to the response |
| `collectionName` | string |  |  | Collection name for vector store |
| `storageServiceEndpoint` | string |  |  | Storage service endpoint URL |
| `storageServiceAuth` | object |  |  | Storage service authentication credentials |
| `embeddingModel` | string |  |  | Embedding model name |
| `embeddingModelProvider` | string |  |  | Embedding model provider ID |
| `embeddingModelDimensions` | integer (int32) |  |  | Embedding model vector dimensions |
| `providerRefId` | string |  |  | Provider reference ID for retrieving credentials from vault |
| `chunkSize` | integer (int32) |  |  | Chunk size for text splitting |
| `chunkOverlap` | integer (int32) |  |  | Chunk overlap for text splitting |
| `referencesFormat` | string |  |  | References format |
| `connectionConfig` | object |  |  | Connection configuration |
| `hydeRetrieval` | boolean | yes |  | Enable HyDE (Hypothetical Document Embeddings) retrieval |


<a id="schema-languagedto"></a>
#### `LanguageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `code` | string | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-llmconfig"></a>
#### `LlmConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `agent_id` | string |  |  |  |


<a id="schema-llmdto"></a>
#### `LlmDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `key` | string | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-llmmodeldto"></a>
#### `LlmModelDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `provider` | string |  |  |  |
| `description` | string |  |  |  |
| `contextLength` | integer (int32) |  |  |  |
| `maxOutputTokens` | integer (int32) |  |  |  |
| `fixedTemperature` | number (double) |  |  |  |
| `pricing` | [`PricingDto`](#schema-pricingdto) |  |  |  |
| `custom` | boolean |  |  |  |
| `embedding` | boolean | yes |  |  |


<a id="schema-llmproviderconfigdto"></a>
#### `LlmProviderConfigDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `providerId` | string | yes |  |  |
| `providerName` | string | yes |  |  |
| `providerRefId` | string | yes |  |  |
| `name` | string | yes |  |  |
| `env` | object | yes |  |  |
| `config` | object | yes |  |  |
| `validationStatus` | string | yes | `VALID`, `INVALID`, `NOT_VALIDATED` |  |
| `lastValidatedAt` | string (date-time) |  |  |  |
| `validationError` | string |  |  |  |
| `system` | boolean | yes |  |  |
| `createdBy` | string | yes |  |  |
| `createdAt` | string (date-time) | yes |  |  |
| `updatedAt` | string (date-time) | yes |  |  |


<a id="schema-llmproviderconfigrequest"></a>
#### `LlmProviderConfigRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `providerId` | string | yes |  |  |
| `providerName` | string | yes |  |  |
| `env` | object |  |  |  |
| `config` | object | yes |  |  |


<a id="schema-llmproviderconfigusagedto"></a>
#### `LlmProviderConfigUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `configId` | integer (int64) | yes |  |  |
| `configName` | string | yes |  |  |
| `agents` | [`LlmProviderConfigUsageItemDto`](#schema-llmproviderconfigusageitemdto)[] | yes |  |  |
| `teams` | [`LlmProviderConfigUsageItemDto`](#schema-llmproviderconfigusageitemdto)[] | yes |  |  |
| `storageResources` | [`LlmProviderConfigUsageItemDto`](#schema-llmproviderconfigusageitemdto)[] | yes |  |  |


<a id="schema-llmproviderconfigusageitemdto"></a>
#### `LlmProviderConfigUsageItemDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `refId` | string |  |  |  |
| `name` | string | yes |  |  |
| `projectId` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |


<a id="schema-llmproviderconfigwithmodelsresponse"></a>
#### `LlmProviderConfigWithModelsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `providerId` | string | yes |  |  |
| `providerRefId` | string | yes |  |  |
| `providerName` | string | yes |  |  |
| `name` | string | yes |  |  |
| `availableModels` | [`LlmModelDto`](#schema-llmmodeldto)[] | yes |  |  |
| `validationStatus` | string | yes | `VALID`, `INVALID`, `NOT_VALIDATED` |  |
| `lastValidatedAt` | string (date-time) |  |  |  |
| `validationError` | string |  |  |  |
| `system` | boolean | yes |  |  |


<a id="schema-llmproviderenvrequest"></a>
#### `LlmProviderEnvRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  |  |
| `env` | object | yes |  |  |


<a id="schema-llmproviderenvvar"></a>
#### `LlmProviderEnvVar`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string | yes |  |  |
| `required` | boolean | yes |  |  |


<a id="schema-llmprovideroptiondto"></a>
#### `LlmProviderOptionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `type` | string | yes | `cloud`, `native`, `aggregator`, `local` |  |
| `env` | [`LlmProviderEnvVar`](#schema-llmproviderenvvar)[] | yes |  |  |


<a id="schema-loginrequest"></a>
#### `LoginRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `username` | string | yes |  |  |
| `password` | string | yes |  |  |


<a id="schema-loginresponsedto"></a>
#### `LoginResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `accessToken` | string | yes |  |  |
| `refreshToken` | string | yes |  |  |
| `expirationTime` | string (date-time) | yes |  |  |


<a id="schema-mcpauthconfig"></a>
#### `MCPAuthConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `token` | string |  |  |  |
| `customHeaders` | [`CustomHeader`](#schema-customheader)[] |  |  |  |


<a id="schema-mcphealthresponse"></a>
#### `MCPHealthResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `mcpServerId` | string | yes |  |  |
| `status` | string | yes |  |  |
| `healthData` | object | yes |  |  |
| `lastChecked` | string |  |  |  |
| `errorMessage` | string |  |  |  |
| `responseTimeMs` | integer (int32) |  |  |  |


<a id="schema-mcpservercreaterequest"></a>
#### `MCPServerCreateRequest`

> Request to create a new MCP server

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string | yes | `stdio`, `streamable-http`, `sse` | Transport type. Use 'stdio' for process-based MCP servers (requires mcp-proxy), 'streamable-http' for HTTP servers, or 'sse' for Server-Sent Events |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports (required for streamable-http and sse). Not used for stdio transport. |
| `baseCommand` | string |  |  | Base command for stdio transport (required for stdio). Format: 'command arg1 arg2 ...'. Supports quoted arguments with spaces (both double and single quotes). Example: 'echo "hello world" test' will parse as command='echo' args=['hello world', 'test'] |
| `defaultEnvVars` | object |  |  | Default environment variables for stdio transport |
| `defaultTimeoutSeconds` | integer (int32) | yes |  | Default timeout in seconds |
| `authType` | string | yes | `none`, `bearer`, `api_key` ; minLength=`1` | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean | yes |  | Whether the server is enabled |
| `metadataSchema` | object |  |  | JSON Schema defining metadata structure for tools. Must have 'type' and 'properties' fields. |
| `parameterMappings` | [`ParameterMapping`](#schema-parametermapping)[] |  |  | Parameter mappings for MCP tools |


<a id="schema-mcpserverresponse"></a>
#### `MCPServerResponse`

> MCP server information

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Internal database ID |
| `aiServiceUuid` | string | yes |  | AI service UUID |
| `organizationId` | integer (int64) | yes |  | Organization ID |
| `projectId` | integer (int64) | yes |  | Project ID |
| `name` | string | yes |  | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string | yes | `stdio`, `streamable-http`, `sse` | Transport type |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports |
| `baseCommand` | string |  |  | Base command for stdio transport |
| `defaultEnvVars` | object |  |  | Default environment variables |
| `defaultTimeoutSeconds` | integer (int32) | yes |  | Default timeout in seconds |
| `authType` | string | yes |  | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean | yes |  | Whether the server is enabled |
| `created` | string (date-time) | yes |  | Creation timestamp |
| `createdBy` | string | yes |  | Created by user |
| `updated` | string (date-time) | yes |  | Last update timestamp |
| `updatedBy` | string | yes |  | Updated by user |
| `toolsCount` | integer (int32) |  |  | Number of tools available from the server |
| `healthStatus` | string |  |  | Health status from AI service |
| `lastChecked` | string (date-time) |  |  | Last health check timestamp |
| `responseTimeMs` | integer (int32) |  |  | Response time in milliseconds |
| `errorMessage` | string |  |  | Error message if unhealthy |
| `metadataSchema` | object |  |  | Metadata schema for tools |
| `parameterMappings` | [`ParameterMapping`](#schema-parametermapping)[] |  |  | Parameter mappings for MCP tools |


<a id="schema-mcpservertoolsresponse"></a>
#### `MCPServerToolsResponse`

> MCP server tools list

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `tools` | [`MCPTool`](#schema-mcptool)[] | yes |  | List of available tools |


<a id="schema-mcpserverupdaterequest"></a>
#### `MCPServerUpdateRequest`

> Request to update an existing MCP server

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`1` ; maxLength=`255` | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string |  | `stdio`, `streamable-http`, `sse` | Transport type. Use 'stdio' for process-based MCP servers (requires mcp-proxy), 'streamable-http' for HTTP servers, or 'sse' for Server-Sent Events |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports (required for streamable-http and sse). Not used for stdio transport. |
| `baseCommand` | string |  |  | Base command for stdio transport (required for stdio). Format: 'command arg1 arg2 ...'. Supports quoted arguments with spaces (both double and single quotes). Updating this triggers mcp-proxy re-orchestration. |
| `defaultEnvVars` | object |  |  | Default environment variables for stdio transport |
| `defaultTimeoutSeconds` | integer (int32) |  |  | Default timeout in seconds |
| `authType` | string |  | `none`, `bearer`, `api_key` | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean |  |  | Whether the server is enabled |
| `metadataSchema` | object |  |  | JSON Schema defining metadata structure for tools. Must have 'type' and 'properties' fields. |
| `parameterMappings` | [`ParameterMapping`](#schema-parametermapping)[] |  |  | Parameter mappings for MCP tools |


<a id="schema-mcpserverwithtoolsresponse"></a>
#### `MCPServerWithToolsResponse`

> MCP server response with tools list

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Internal database ID |
| `aiServiceUuid` | string | yes |  | AI service UUID |
| `organizationId` | integer (int64) | yes |  | Organization ID |
| `projectId` | integer (int64) | yes |  | Project ID |
| `name` | string | yes |  | MCP server name |
| `description` | string |  |  | MCP server description |
| `transport` | string | yes | `stdio`, `streamable-http`, `sse` | Transport type |
| `baseUrl` | string |  |  | Base URL for HTTP/SSE transports |
| `baseCommand` | string |  |  | Base command for stdio transport |
| `defaultEnvVars` | object |  |  | Default environment variables |
| `defaultTimeoutSeconds` | integer (int32) | yes |  | Default timeout in seconds |
| `authType` | string | yes |  | Authentication type |
| `authConfig` | [`MCPAuthConfig`](#schema-mcpauthconfig) |  |  | Authentication configuration |
| `enabled` | boolean | yes |  | Whether the server is enabled |
| `created` | string (date-time) | yes |  | Creation timestamp |
| `createdBy` | string | yes |  | Created by user |
| `updated` | string (date-time) | yes |  | Last update timestamp |
| `updatedBy` | string | yes |  | Updated by user |
| `toolsCount` | integer (int32) |  |  | Number of tools available from the server |
| `healthStatus` | string |  |  | Health status from AI service |
| `lastChecked` | string (date-time) |  |  | Last health check timestamp |
| `responseTimeMs` | integer (int32) |  |  | Response time in milliseconds |
| `errorMessage` | string |  |  | Error message if unhealthy |
| `tools` | string[] | yes |  | List of tool names available on this server |


<a id="schema-mcptool"></a>
#### `MCPTool`

> MCP tool information

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  | Tool name |
| `description` | string |  |  | Tool description |
| `parameters` | object |  |  | Tool parameters |


<a id="schema-mcpconfiguration"></a>
#### `McpConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `mcpServerId` | string | yes |  | MCP server ID |
| `mcpServerName` | string | yes |  | MCP server name |
| `enabledTools` | string[] |  |  | List of enabled tools for this MCP server |
| `enabled` | boolean | yes |  | Whether this MCP server is enabled |


<a id="schema-memoryconfiguration"></a>
#### `MemoryConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `clearMemories` | boolean |  |  |  |
| `debugMode` | boolean |  |  |  |
| `deleteMemories` | boolean |  |  |  |
| `enableAgenticMemory` | boolean |  |  |  |
| `enableSessionSummaries` | boolean |  |  |  |
| `enableUserMemories` | boolean |  |  | Use updateMemoryOnRun instead |
| `maxMemorySize` | integer (int32) |  |  |  |
| `memoryRetrievalLimit` | integer (int32) |  |  |  |
| `retentionDays` | integer (int32) |  |  |  |
| `storageConfig` | object |  |  |  |
| `updateMemoryOnRun` | boolean |  |  | Update memory on each run (replaces enable_user_memories) |
| `searchSessionHistory` | boolean |  |  | Search through session history |
| `numHistorySessions` | integer (int32) |  | minimum=`0` ; maximum=`50` | Number of history sessions to include |


<a id="schema-memorylistdto"></a>
#### `MemoryListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `memoryId` | string | yes |  |  |
| `memory` | string | yes |  |  |
| `topics` | string[] | yes |  |  |
| `agentId` | string |  |  |  |
| `teamId` | string |  |  |  |
| `userId` | string |  |  |  |
| `updatedAt` | string (date-time) |  |  |  |


<a id="schema-metadataschemafield"></a>
#### `MetadataSchemaField`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `type` | string | yes |  |  |
| `required` | boolean | yes |  |  |
| `description` | string |  |  |  |


<a id="schema-metrics"></a>
#### `Metrics`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `inputTokens` | integer (int32) |  |  |  |
| `outputTokens` | integer (int32) |  |  |  |
| `totalTokens` | integer (int32) |  |  |  |
| `duration` | number (double) |  |  |  |
| `toolCallCount` | integer (int32) |  |  |  |
| `ragRetrievalCount` | integer (int32) |  |  |  |
| `documentPageCount` | integer (int32) |  |  |  |


<a id="schema-modelconfigurationlong"></a>
#### `ModelConfigurationLong`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | integer (int64) | yes |  | Provider identifier (Long for database, String for AI service) |
| `modelName` | string | yes |  | Model name |
| `temperature` | number (double) |  | minimum=`0` ; maximum=`2` |  |
| `maxTokens` | integer (int32) |  |  |  |
| `frequencyPenalty` | number (double) |  |  |  |
| `presencePenalty` | number (double) |  |  |  |
| `topP` | number (double) |  |  |  |
| `providerRefId` | string |  |  |  |
| `providerName` | string |  |  |  |
| `reasoningConfig` | object |  |  |  |
| `failoverConfig` | [`FailoverModelConfigLong`](#schema-failovermodelconfiglong)[] |  |  | Ordered list of fallback model configurations used when the primary model fails |


<a id="schema-modelconfigurationstring"></a>
#### `ModelConfigurationString`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes |  | Provider identifier (Long for database, String for AI service) |
| `modelName` | string | yes |  | Model name |
| `temperature` | number (double) |  | minimum=`0` ; maximum=`2` |  |
| `maxTokens` | integer (int32) |  |  |  |
| `frequencyPenalty` | number (double) |  |  |  |
| `presencePenalty` | number (double) |  |  |  |
| `topP` | number (double) |  |  |  |
| `providerRefId` | string |  |  |  |
| `providerName` | string |  |  |  |
| `reasoningConfig` | object |  |  |  |
| `failoverConfig` | [`FailoverModelConfigString`](#schema-failovermodelconfigstring)[] |  |  | Ordered list of fallback model configurations used when the primary model fails |


<a id="schema-multipartlimitsdto"></a>
#### `MultipartLimitsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `maxFileSize` | [`ValueUnitDto`](#schema-valueunitdto) | yes |  |  |
| `maxRequestSize` | [`ValueUnitDto`](#schema-valueunitdto) | yes |  |  |


<a id="schema-n8ndomainresponse"></a>
#### `N8nDomainResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `domain` | string |  |  |  |


<a id="schema-nextaction"></a>
#### `NextAction`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `type` | string | yes |  |  |
| `clientSecret` | string |  |  |  |


<a id="schema-organizationdeploymentresponse"></a>
#### `OrganizationDeploymentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `deploymentId` | integer (int64) | yes |  |  |
| `status` | string | yes |  |  |
| `subdomain` | string |  |  |  |
| `accessUrl` | string | yes |  |  |
| `sshUsername` | string | yes |  |  |
| `n8nUsername` | string | yes |  |  |
| `serverIp` | string | yes |  |  |
| `port` | integer (int32) | yes |  |  |
| `deploymentWorkerId` | string |  |  |  |
| `errorMessage` | string |  |  |  |
| `createdAt` | string (date-time) | yes |  |  |
| `updatedAt` | string (date-time) | yes |  |  |
| `completedAt` | string (date-time) |  |  |  |


<a id="schema-organizationdto"></a>
#### `OrganizationDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) |  |  |  |
| `name` | string |  |  |  |
| `referenceId` | string |  |  |  |
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) |  |  |  |
| `projects` | [`ProjectDto`](#schema-projectdto)[] | yes |  |  |
| `created` | string (date-time) |  |  |  |
| `updated` | string (date-time) |  |  |  |
| `createdBy` | [`BaseUserDto`](#schema-baseuserdto) |  |  |  |
| `isMember` | boolean | yes |  |  |
| `member` | boolean |  |  |  |


<a id="schema-organizationlimitsdto"></a>
#### `OrganizationLimitsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `rateLimits` | [`RateLimitsDto`](#schema-ratelimitsdto) | yes |  |  |
| `fileUpload` | [`FileUploadLimitsDto`](#schema-fileuploadlimitsdto) | yes |  |  |


<a id="schema-organizationlistdto"></a>
#### `OrganizationListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `referenceId` | string | yes |  |  |
| `memberCount` | integer (int32) | yes |  |  |
| `projectCount` | integer (int32) | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `createdBy` | [`BaseUserDto`](#schema-baseuserdto) |  |  |  |


<a id="schema-organizationmemberdto"></a>
#### `OrganizationMemberDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `email` | string | yes |  |  |
| `permissions` | [`OrganizationPermissions`](#schema-organizationpermissions) | yes |  |  |
| `status` | string | yes | `PENDING`, `ACTIVE`, `BLOCKED` |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-organizationpermissions"></a>
#### `OrganizationPermissions`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `log` | [`Permission`](#schema-permission) | yes |  |  |
| `tool` | [`Permission`](#schema-permission) | yes |  |  |
| `skill` | [`Permission`](#schema-permission) | yes |  |  |
| `audio` | [`Permission`](#schema-permission) | yes |  |  |
| `team` | [`Permission`](#schema-permission) | yes |  |  |
| `agent` | [`Permission`](#schema-permission) | yes |  |  |
| `project` | [`Permission`](#schema-permission) | yes |  |  |
| `workflow` | [`Permission`](#schema-permission) | yes |  |  |
| `knowledge` | [`Permission`](#schema-permission) | yes |  |  |
| `playground` | [`Permission`](#schema-permission) | yes |  |  |
| `playground_session` | [`Permission`](#schema-permission) | yes |  |  |
| `policy` | [`Permission`](#schema-permission) | yes |  |  |
| `member` | [`Permission`](#schema-permission) | yes |  |  |
| `evaluation` | [`Permission`](#schema-permission) | yes |  |  |
| `webhook` | [`Permission`](#schema-permission) | yes |  |  |
| `api_key` | [`Permission`](#schema-permission) | yes |  |  |
| `organization` | [`Permission`](#schema-permission) | yes |  |  |
| `billing` | [`Permission`](#schema-permission) | yes |  |  |
| `data_usage` | [`Permission`](#schema-permission) | yes |  |  |
| `llm_provider` | [`Permission`](#schema-permission) | yes |  |  |
| `api_reference` | [`Permission`](#schema-permission) | yes |  |  |
| `secrets` | [`Permission`](#schema-permission) | yes |  |  |


<a id="schema-organizationrequest"></a>
#### `OrganizationRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |


<a id="schema-organizationsetupdto"></a>
#### `OrganizationSetupDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKeyName` | string | yes |  |  |
| `projectName` | string | yes |  |  |


<a id="schema-organizationusagesummarydto"></a>
#### `OrganizationUsageSummaryDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalRequests` | integer (int64) | yes |  |  |
| `successCount` | integer (int64) | yes |  |  |
| `successRate` | number | yes |  |  |
| `totalInputTokens` | integer (int64) | yes |  |  |
| `totalOutputTokens` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |
| `creditBalance` | integer (int64) | yes |  |  |
| `usage` | [`UsageDto`](#schema-usagedto)[] | yes |  |  |
| `topProjects` | [`TopProjectUsageDto`](#schema-topprojectusagedto)[] | yes |  |  |
| `topUsers` | [`TopUserUsageDto`](#schema-topuserusagedto)[] | yes |  |  |
| `topAgents` | [`TopComponentUsageDto`](#schema-topcomponentusagedto)[] | yes |  |  |
| `topApiKeys` | [`TopApiKeyUsageDto`](#schema-topapikeyusagedto)[] | yes |  |  |


<a id="schema-outboundcallresponsedto"></a>
#### `OutboundCallResponseDto`

> Outbound call response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `roomName` | string | yes |  | LiveKit room name for this call |
| `participantId` | string | yes |  | SIP participant ID |
| `toNumber` | string | yes |  | Phone number being called |
| `fromNumber` | string | yes |  | Phone number calling from |
| `voicePresetId` | string | yes |  | Voice preset ID used for this call |
| `userId` | string | yes |  | External user identifier |
| `sessionId` | string | yes |  | AI Service session ID |


<a id="schema-outboundconfigdto"></a>
#### `OutboundConfigDto`

> Outbound telephony configuration

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `ip` | string | yes |  | SIP server address (hostname or IP). For Twilio use: your-trunk.pstn.twilio.com |
| `transport` | string | yes | `SIP_TRANSPORT_AUTO`, `SIP_TRANSPORT_UDP`, `SIP_TRANSPORT_TCP`, `SIP_TRANSPORT_TLS` | SIP transport protocol |
| `numbers` | [`OutboundPhoneConfigDto`](#schema-outboundphoneconfigdto)[] |  |  | List of outbound phone numbers with voice presets |
| `mediaEncryption` | string | yes | `SIP_MEDIA_ENCRYPT_DISABLE`, `SIP_MEDIA_ENCRYPT_ALLOW`, `SIP_MEDIA_ENCRYPT_REQUIRE` | Media encryption mode |
| `username` | string |  |  | SIP username for authentication |
| `password` | string |  |  | SIP password for authentication |


<a id="schema-outboundphoneconfigdto"></a>
#### `OutboundPhoneConfigDto`

> Outbound phone number configuration with voice preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `number` | string | yes | minLength=`1` ; pattern=`^\+?[0-9]+$` | Phone number (digits only, optional + prefix) |
| `voicePresetId` | string | yes | minLength=`1` | Voice preset ID to use for this number |


<a id="schema-page"></a>
#### `Page`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | object[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageagentresponse"></a>
#### `PageAgentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`AgentResponse`](#schema-agentresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageagentrundto"></a>
#### `PageAgentRunDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`AgentRunDto`](#schema-agentrundto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageagentversionresponse"></a>
#### `PageAgentVersionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`AgentVersionResponse`](#schema-agentversionresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageapikeydto"></a>
#### `PageApiKeyDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`ApiKeyDto`](#schema-apikeydto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagedeploymentlogdto"></a>
#### `PageDeploymentLogDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`DeploymentLogDto`](#schema-deploymentlogdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageevalrunlistdto"></a>
#### `PageEvalRunListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`EvalRunListDto`](#schema-evalrunlistdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagehttprequestresponse"></a>
#### `PageHttpRequestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`HttpRequestResponse`](#schema-httprequestresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagellmproviderconfigdto"></a>
#### `PageLlmProviderConfigDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`LlmProviderConfigDto`](#schema-llmproviderconfigdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagememorylistdto"></a>
#### `PageMemoryListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`MemoryListDto`](#schema-memorylistdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageorganizationlistdto"></a>
#### `PageOrganizationListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`OrganizationListDto`](#schema-organizationlistdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagepaymenttransactiondto"></a>
#### `PagePaymentTransactionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`PaymentTransactionDto`](#schema-paymenttransactiondto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagepolicyauditlogdto"></a>
#### `PagePolicyAuditLogDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`PolicyAuditLogDTO`](#schema-policyauditlogdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagepolicyitemdto"></a>
#### `PagePolicyItemDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`PolicyItemDTO`](#schema-policyitemdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagesecretresponse"></a>
#### `PageSecretResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`SecretResponse`](#schema-secretresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagesessionlistdto"></a>
#### `PageSessionListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`SessionListDto`](#schema-sessionlistdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageskillresponse"></a>
#### `PageSkillResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`SkillResponse`](#schema-skillresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageteamresponse"></a>
#### `PageTeamResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`TeamResponse`](#schema-teamresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagetelephonypresetresponsedto"></a>
#### `PageTelephonyPresetResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagetracelistdto"></a>
#### `PageTraceListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`TraceListDto`](#schema-tracelistdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagetracesessionstatsdto"></a>
#### `PageTraceSessionStatsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`TraceSessionStatsDto`](#schema-tracesessionstatsdto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagevoicepresetsummarydto"></a>
#### `PageVoicePresetSummaryDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`VoicePresetSummaryDto`](#schema-voicepresetsummarydto)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagewebhookdeliveryresponse"></a>
#### `PageWebhookDeliveryResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`WebhookDeliveryResponse`](#schema-webhookdeliveryresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pagewebhooksubscriptionresponse"></a>
#### `PageWebhookSubscriptionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalElements` | integer (int64) |  |  |  |
| `totalPages` | integer (int32) |  |  |  |
| `first` | boolean |  |  |  |
| `last` | boolean |  |  |  |
| `size` | integer (int32) |  |  |  |
| `content` | [`WebhookSubscriptionResponse`](#schema-webhooksubscriptionresponse)[] |  |  |  |
| `number` | integer (int32) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `numberOfElements` | integer (int32) |  |  |  |
| `pageable` | [`PageableObject`](#schema-pageableobject) |  |  |  |
| `empty` | boolean |  |  |  |


<a id="schema-pageable"></a>
#### `Pageable`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `page` | integer (int32) |  | minimum=`0` |  |
| `size` | integer (int32) |  | minimum=`1` |  |
| `sort` | string[] |  |  |  |


<a id="schema-pageableobject"></a>
#### `PageableObject`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `offset` | integer (int64) |  |  |  |
| `sort` | [`SortObject`](#schema-sortobject) |  |  |  |
| `paged` | boolean |  |  |  |
| `pageNumber` | integer (int32) |  |  |  |
| `pageSize` | integer (int32) |  |  |  |
| `unpaged` | boolean |  |  |  |


<a id="schema-pagedresponseagentrundto"></a>
#### `PagedResponseAgentRunDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`AgentRunDto`](#schema-agentrundto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponseapikeydto"></a>
#### `PagedResponseApiKeyDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`ApiKeyDto`](#schema-apikeydto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponseevalrunlistdto"></a>
#### `PagedResponseEvalRunListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`EvalRunListDto`](#schema-evalrunlistdto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponsesessionlistdto"></a>
#### `PagedResponseSessionListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SessionListDto`](#schema-sessionlistdto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponseskillresponse"></a>
#### `PagedResponseSkillResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`SkillResponse`](#schema-skillresponse)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponsetelephonypresetresponsedto"></a>
#### `PagedResponseTelephonyPresetResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TelephonyPresetResponseDto`](#schema-telephonypresetresponsedto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponsetracelistdto"></a>
#### `PagedResponseTraceListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TraceListDto`](#schema-tracelistdto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-pagedresponsetracesessionstatsdto"></a>
#### `PagedResponseTraceSessionStatsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `success` | boolean | yes |  |  |
| `data` | [`TraceSessionStatsDto`](#schema-tracesessionstatsdto)[] | yes |  |  |
| `pagination` | [`PaginationInfo`](#schema-paginationinfo) | yes |  |  |
| `message` | string |  |  |  |
| `timestamp` | string (date-time) | yes |  |  |
| `version` | string | yes |  |  |


<a id="schema-paginationinfo"></a>
#### `PaginationInfo`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `page` | integer (int32) | yes |  |  |
| `size` | integer (int32) | yes |  |  |
| `totalElements` | integer (int64) | yes |  |  |
| `totalPages` | integer (int32) | yes |  |  |
| `hasNext` | boolean | yes |  |  |
| `hasPrevious` | boolean | yes |  |  |


<a id="schema-parametermapping"></a>
#### `ParameterMapping`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `toolName` | string | yes |  |  |
| `toolParameter` | string | yes |  |  |
| `metadataKey` | string | yes |  |  |
| `transformationType` | string | yes |  |  |
| `transformationConfig` | string |  |  |  |
| `enabled` | boolean | yes |  |  |


<a id="schema-paymentmethoddto"></a>
#### `PaymentMethodDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) |  |  |  |
| `stripePaymentMethodId` | string |  |  |  |
| `type` | string |  |  |  |
| `lastFour` | string |  |  |  |
| `expirationMonth` | integer (int32) |  |  |  |
| `expirationYear` | integer (int32) |  |  |  |
| `brand` | string |  |  |  |
| `isDefault` | boolean | yes |  |  |
| `default` | boolean |  |  |  |


<a id="schema-paymenttransactiondto"></a>
#### `PaymentTransactionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `amount` | integer (int64) | yes |  |  |
| `currency` | string | yes | `USD`, `EUR` |  |
| `referenceId` | string | yes |  |  |
| `type` | string | yes | `PAYMENT`, `REFUND`, `CHARGE`, `ADJUSTMENT`, `AUTO_RECHARGE` |  |
| `description` | string | yes |  |  |
| `balanceAfter` | integer (int64) | yes |  |  |
| `status` | string | yes | `PENDING`, `COMPLETED`, `FAILED`, `REFUNDED` |  |
| `stripePaymentIntentId` | string |  |  |  |
| `stripeChargeId` | string |  |  |  |
| `error` | string |  |  |  |
| `receiptUrl` | string |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-permission"></a>
#### `Permission`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `read` | boolean |  |  |  |
| `write` | boolean |  |  |  |
| `delete` | boolean |  |  |  |
| `category` | string | yes | `PLATFORM`, `DOCUMENTATION`, `SETTINGS_PROJECT`, `SETTINGS_ORGANIZATION` |  |


<a id="schema-playgroundrunresponse"></a>
#### `PlaygroundRunResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `session_id` | string | yes |  |  |


<a id="schema-policyauditlogdto"></a>
#### `PolicyAuditLogDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `policyReferenceId` | string | yes |  |  |
| `policyName` | string | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `projectId` | integer (int64) |  |  |  |
| `teamId` | integer (int64) |  |  |  |
| `agentId` | integer (int64) |  |  |  |
| `callerReferenceId` | string |  |  |  |
| `callerType` | string |  | `USER`, `API_KEY`, `WEBHOOK`, `SERVICE_ACCOUNT`, `UNKNOWN` |  |
| `sessionId` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `detectionMethod` | string | yes | `REGEX` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `confidenceScore` | number (double) |  |  |  |
| `contextHash` | string | yes |  |  |
| `processingTimeMs` | integer (int64) | yes |  |  |
| `created` | string (date-time) | yes |  |  |


<a id="schema-policyauditlogstatistics"></a>
#### `PolicyAuditLogStatistics`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalLogs` | integer (int64) | yes |  |  |
| `averageProcessingTimeMs` | number (double) | yes |  |  |
| `logsByAction` | object | yes |  |  |
| `logsByStage` | object | yes |  |  |
| `periodStart` | string (date-time) | yes |  |  |
| `periodEnd` | string (date-time) | yes |  |  |


<a id="schema-policyenforcementdto"></a>
#### `PolicyEnforcementDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `policyId` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `projectId` | integer (int64) |  |  |  |
| `teamId` | integer (int64) |  |  |  |
| `agentId` | integer (int64) |  |  |  |
| `enabled` | boolean | yes |  |  |
| `scope` | string | yes | `ORG`, `PROJECT`, `TEAM`, `AGENT` |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-policyitemdto"></a>
#### `PolicyItemDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `referenceId` | string | yes |  |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes |  |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `isDefault` | boolean | yes |  |  |
| `organizationId` | integer (int64) |  |  |  |
| `frameworkIds` | integer (int64)[] | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `enabled` | boolean |  |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |


<a id="schema-policyitemforevaluation"></a>
#### `PolicyItemForEvaluation`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `reference_id` | string | yes |  |  |
| `type` | string | yes |  |  |
| `regex` | string | yes |  |  |
| `action` | string | yes |  |  |
| `stage` | string | yes |  |  |
| `name` | string | yes |  |  |
| `redact_config` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `block_config` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alert_config` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transform_config` | [`TransformConfig`](#schema-transformconfig) |  |  |  |


<a id="schema-policymodel"></a>
#### `PolicyModel`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` |  |
| `description` | string |  |  |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `frameworks` | integer (int64)[] | yes |  |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |


<a id="schema-policystatisticdto"></a>
#### `PolicyStatisticDTO`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `policyReferenceId` | string | yes |  |  |
| `policyName` | string | yes |  |  |
| `triggerCount` | integer (int64) | yes |  |  |
| `lastTriggeredAt` | string (date-time) |  |  |  |


<a id="schema-policystatisticsresponse"></a>
#### `PolicyStatisticsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `summary` | [`PolicyStatisticsSummary`](#schema-policystatisticssummary) | yes |  |  |
| `policies` | [`PolicyStatisticDTO`](#schema-policystatisticdto)[] | yes |  |  |


<a id="schema-policystatisticssummary"></a>
#### `PolicyStatisticsSummary`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `count` | integer (int32) | yes |  |  |
| `totalTriggers` | integer (int64) | yes |  |  |


<a id="schema-policytestrequest"></a>
#### `PolicyTestRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `text` | string | yes | minLength=`1` |  |
| `stage` | string | yes | `PRE_PROMPT`, `POST_RESPONSE` |  |
| `type` | string | yes | `REGEX` |  |
| `regex` | string | yes | minLength=`1` |  |
| `action` | string | yes | `REDACT`, `BLOCK`, `TRANSFORM`, `ALERT` |  |
| `redactConfig` | [`RedactConfig`](#schema-redactconfig) |  |  |  |
| `blockConfig` | [`BlockConfig`](#schema-blockconfig) |  |  |  |
| `transformConfig` | [`TransformConfig`](#schema-transformconfig) |  |  |  |
| `alertConfig` | [`AlertConfig`](#schema-alertconfig) |  |  |  |


<a id="schema-policytestresponse"></a>
#### `PolicyTestResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `originalText` | string | yes |  |  |
| `processedText` | string | yes |  |  |
| `matched` | boolean | yes |  |  |
| `action` | string | yes |  |  |
| `matchCount` | integer (int32) | yes |  |  |
| `blocked` | boolean | yes |  |  |
| `message` | string |  |  |  |


<a id="schema-pricingdto"></a>
#### `PricingDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `prompt` | number |  |  |  |
| `completion` | number |  |  |  |
| `image` | number |  |  |  |
| `request` | number |  |  |  |
| `inputCacheReads` | number |  |  |  |
| `inputCacheWrites` | number |  |  |  |


<a id="schema-projectdto"></a>
#### `ProjectDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) |  |  |  |
| `referenceId` | string |  |  |  |
| `name` | string |  |  |  |
| `isDefault` | boolean |  |  |  |
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |
| `membersCount` | integer (int32) |  |  |  |
| `created` | string (date-time) |  |  |  |
| `updated` | string (date-time) |  |  |  |
| `archived` | string (date-time) |  |  |  |
| `createdBy` | [`BaseUserDto`](#schema-baseuserdto) |  |  |  |
| `archivedBy` | [`BaseUserDto`](#schema-baseuserdto) |  |  |  |


<a id="schema-projectmemberdto"></a>
#### `ProjectMemberDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `refId` | string | yes |  |  |
| `email` | string | yes |  |  |
| `permissions` | [`ProjectPermissions`](#schema-projectpermissions) | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |
| `status` | string | yes | `PENDING`, `ACTIVE`, `BLOCKED` |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-projectoptiondto"></a>
#### `ProjectOptionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `apiKeys` | [`ApiKeyOptionDto`](#schema-apikeyoptiondto)[] | yes |  |  |


<a id="schema-projectpermissions"></a>
#### `ProjectPermissions`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `log` | [`Permission`](#schema-permission) | yes |  |  |
| `tool` | [`Permission`](#schema-permission) | yes |  |  |
| `skill` | [`Permission`](#schema-permission) | yes |  |  |
| `audio` | [`Permission`](#schema-permission) | yes |  |  |
| `team` | [`Permission`](#schema-permission) | yes |  |  |
| `agent` | [`Permission`](#schema-permission) | yes |  |  |
| `project` | [`Permission`](#schema-permission) | yes |  |  |
| `member` | [`Permission`](#schema-permission) | yes |  |  |
| `evaluation` | [`Permission`](#schema-permission) | yes |  |  |
| `webhook` | [`Permission`](#schema-permission) | yes |  |  |
| `workflow` | [`Permission`](#schema-permission) | yes |  |  |
| `knowledge` | [`Permission`](#schema-permission) | yes |  |  |
| `playground` | [`Permission`](#schema-permission) | yes |  |  |
| `playground_session` | [`Permission`](#schema-permission) | yes |  |  |
| `policy` | [`Permission`](#schema-permission) | yes |  |  |


<a id="schema-projectusagedto"></a>
#### `ProjectUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `referenceId` | string | yes |  |  |
| `name` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `avgSuccessRate` | number | yes |  |  |
| `totalStorageBytes` | integer (int64) | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `keysUsage` | [`KeyUsageDto`](#schema-keyusagedto)[] | yes |  |  |
| `usersUsage` | [`UserUsageDto`](#schema-userusagedto)[] | yes |  |  |
| `agentsUsage` | [`AgentUsageDtoObject`](#schema-agentusagedtoobject)[] | yes |  |  |


<a id="schema-ratelimitconfigrequestdto"></a>
#### `RateLimitConfigRequestDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `requestsPerMinute` | integer (int64) | yes |  |  |
| `requestsPerHour` | integer (int64) | yes |  |  |
| `requestsPerDay` | integer (int64) | yes |  |  |
| `burstCapacity` | integer (int64) | yes |  |  |


<a id="schema-ratelimitconfigresponsedto"></a>
#### `RateLimitConfigResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `requestsPerMinute` | integer (int64) | yes |  |  |
| `requestsPerHour` | integer (int64) | yes |  |  |
| `requestsPerDay` | integer (int64) | yes |  |  |
| `burstCapacity` | integer (int64) | yes |  |  |
| `isActive` | boolean | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-ratelimitsdto"></a>
#### `RateLimitsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `requestsPerMinute` | integer (int64) | yes |  |  |
| `requestsPerHour` | integer (int64) | yes |  |  |
| `requestsPerDay` | integer (int64) | yes |  |  |
| `burstCapacity` | integer (int64) | yes |  |  |


<a id="schema-reasoningparam"></a>
#### `ReasoningParam`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string | yes |  |  |
| `label` | string | yes |  |  |
| `description` | string | yes |  |  |
| `type` | string | yes |  |  |
| `default` | object |  |  |  |
| `options` | string[] |  |  |  |
| `min` | integer (int32) |  |  |  |
| `max` | integer (int32) |  |  |  |
| `step` | integer (int32) |  |  |  |


<a id="schema-reasoningparamsresponse"></a>
#### `ReasoningParamsResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `supported` | boolean | yes |  |  |
| `params` | [`ReasoningParam`](#schema-reasoningparam)[] | yes |  |  |


<a id="schema-redactconfig"></a>
#### `RedactConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `type` | string | yes | `MASK`, `PRESERVE_LENGTH`, `PRESERVE_FORMAT` |  |
| `mask` | string |  |  |  |


<a id="schema-renamesessionrequestdto"></a>
#### `RenameSessionRequestDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionName` | string | yes |  |  |


<a id="schema-rolepresetcreaterequest"></a>
#### `RolePresetCreateRequest`

> Request to create a new role preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | Role preset name |
| `type` | string | yes | `ORGANIZATION`, `PROJECT` | Role preset type (ORGANIZATION or PROJECT) |
| `permissions` | object | yes |  | Permissions configuration as map - converted to OrganizationPermissions or ProjectPermissions based on type |


<a id="schema-rolepresetresponse"></a>
#### `RolePresetResponse`

> Role preset information

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Role preset ID |
| `organizationId` | integer (int64) | yes |  | Organization ID |
| `name` | string | yes |  | Role preset name |
| `type` | string | yes | `ORGANIZATION`, `PROJECT` | Role preset type (ORGANIZATION or PROJECT) |
| `permissions` | [`SharedPermissions`](#schema-sharedpermissions) | yes |  | Permissions configuration - OrganizationPermissions or ProjectPermissions based on type |
| `createdTs` | string (date-time) | yes |  | Creation timestamp |
| `updatedTs` | string (date-time) | yes |  | Last update timestamp |


<a id="schema-rolepresetupdaterequest"></a>
#### `RolePresetUpdateRequest`

> Request to update an existing role preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`255` | Role preset name |
| `permissions` | object | yes |  | Permissions configuration as map - converted based on preset type |


<a id="schema-rooturldto"></a>
#### `RootUrlDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `isRoot` | boolean | yes |  |  |
| `url` | string | yes |  |  |
| `pageTitle` | string |  |  |  |
| `status` | string | yes |  |  |
| `error` | string |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `children` | [`UrlChildDto`](#schema-urlchilddto)[] |  |  |  |
| `hasFile` | boolean | yes |  |  |
| `urlsWithFilesCount` | integer (int32) |  |  |  |
| `urlsWithoutFilesCount` | integer (int32) |  |  |  |
| `fileSize` | integer (int64) |  |  |  |
| `totalFileSize` | integer (int64) |  |  |  |
| `type` | string | yes |  |  |
| `errorSource` | string |  |  |  |
| `crawlStopReason` | string |  |  |  |
| `chunkingStrategy` | string |  |  |  |
| `chunkingConfig` | object |  |  |  |
| `filterMetadata` | object |  |  |  |
| `ignoreLinks` | boolean | yes |  |  |


<a id="schema-sshpublickeyresponse"></a>
#### `SSHPublicKeyResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `publicKey` | string | yes |  |  |
| `instructions` | string | yes |  |  |
| `addKeyCommand` | string | yes |  |  |


<a id="schema-sttprovidertestrequest"></a>
#### `STTProviderTestRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes | `google`, `soniox`, `azure`, `cartesia`, `eleven`, `deepgram`, `assemblyai` |  |
| `credentials` | object | yes |  |  |


<a id="schema-searchresponse"></a>
#### `SearchResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `query` | string | yes |  |  |
| `results` | [`SearchResult`](#schema-searchresult)[] | yes |  |  |


<a id="schema-searchresult"></a>
#### `SearchResult`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `documentId` | string | yes |  |  |
| `content` | string | yes |  |  |


<a id="schema-secretresponse"></a>
#### `SecretResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `uuid` | string (uuid) | yes |  |  |
| `key` | string | yes |  |  |
| `description` | string |  |  |  |
| `maskedValue` | string | yes |  |  |
| `createdAt` | string (date-time) | yes |  |  |
| `updatedAt` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `updatedBy` | string | yes |  |  |


<a id="schema-serversenteventmapstringobject"></a>
#### `ServerSentEventMapStringObject`

```json
{}
```


<a id="schema-serversenteventstring"></a>
#### `ServerSentEventString`

```json
{}
```


<a id="schema-sessiondto"></a>
#### `SessionDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `userId` | string |  |  |  |
| `sessionId` | string |  |  |  |
| `sessionName` | string |  |  |  |
| `sessionSummary` | [`SessionSummaryDto`](#schema-sessionsummarydto) |  |  |  |
| `sessionState` | object |  |  |  |
| `componentId` | string |  |  |  |
| `totalTokens` | integer (int32) |  |  |  |
| `componentData` | object |  |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `chatHistory` | object[] |  |  |  |
| `createdAt` | string (date-time) |  |  |  |
| `updatedAt` | string (date-time) |  |  |  |


<a id="schema-sessionlistdto"></a>
#### `SessionListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sessionId` | string |  |  |  |
| `sessionName` | string |  |  |  |
| `sessionState` | object |  |  |  |
| `createdAt` | string (date-time) |  |  |  |
| `updatedAt` | string (date-time) |  |  |  |


<a id="schema-sessionrundto"></a>
#### `SessionRunDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `runId` | string |  |  |  |
| `parentRunId` | string |  |  |  |
| `componentId` | string |  |  |  |
| `userId` | string |  |  |  |
| `runInput` | string |  |  |  |
| `content` | object |  |  |  |
| `status` | string |  |  |  |
| `contentType` | string |  |  |  |
| `runResponseFormat` | string |  |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `tools` | object[] |  |  |  |
| `images` | object[] |  |  |  |
| `events` | object[] |  |  |  |
| `messages` | object[] |  |  |  |
| `stepResults` | object[] |  |  |  |
| `stepExecutorRuns` | object[] |  |  |  |
| `createdAt` | string (date-time) |  |  |  |


<a id="schema-sessionstorageconfiguration"></a>
#### `SessionStorageConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `addHistoryToContext` | boolean |  |  |  |
| `readChatHistory` | boolean |  |  |  |
| `numHistoryRuns` | integer (int32) |  |  |  |
| `storageConfig` | object |  |  |  |


<a id="schema-sessionsummarydto"></a>
#### `SessionSummaryDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `summary` | string | yes |  |  |
| `updatedAt` | string (date-time) | yes |  |  |


<a id="schema-setupintentresponse"></a>
#### `SetupIntentResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `clientSecret` | string | yes |  |  |
| `publishableKey` | string | yes |  |  |


<a id="schema-sharedpermissions"></a>
#### `SharedPermissions`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `policy` | [`Permission`](#schema-permission) | yes |  |  |
| `log` | [`Permission`](#schema-permission) | yes |  |  |
| `member` | [`Permission`](#schema-permission) | yes |  |  |
| `audio` | [`Permission`](#schema-permission) | yes |  |  |
| `agent` | [`Permission`](#schema-permission) | yes |  |  |
| `project` | [`Permission`](#schema-permission) | yes |  |  |
| `team` | [`Permission`](#schema-permission) | yes |  |  |
| `skill` | [`Permission`](#schema-permission) | yes |  |  |
| `tool` | [`Permission`](#schema-permission) | yes |  |  |
| `workflow` | [`Permission`](#schema-permission) | yes |  |  |
| `knowledge` | [`Permission`](#schema-permission) | yes |  |  |
| `playground` | [`Permission`](#schema-permission) | yes |  |  |
| `playgroundSession` | [`Permission`](#schema-permission) | yes |  |  |
| `evaluation` | [`Permission`](#schema-permission) | yes |  |  |
| `webhook` | [`Permission`](#schema-permission) | yes |  |  |


<a id="schema-signuprequest"></a>
#### `SignUpRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `firstName` | string |  |  |  |
| `lastName` | string |  |  |  |
| `email` | string | yes |  |  |
| `password` | string | yes |  |  |


<a id="schema-simpletagdto"></a>
#### `SimpleTagDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-singlepageurldto"></a>
#### `SinglePageUrlDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `isRoot` | boolean | yes |  |  |
| `url` | string | yes |  |  |
| `pageTitle` | string |  |  |  |
| `status` | string | yes |  |  |
| `error` | string |  |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `children` | [`UrlChildDto`](#schema-urlchilddto)[] |  |  |  |
| `hasFile` | boolean | yes |  |  |
| `urlsWithFilesCount` | integer (int32) |  |  |  |
| `urlsWithoutFilesCount` | integer (int32) |  |  |  |
| `fileSize` | integer (int64) |  |  |  |
| `totalFileSize` | integer (int64) |  |  |  |
| `type` | string | yes |  |  |
| `errorSource` | string |  |  |  |
| `crawlStopReason` | string |  |  |  |
| `chunkingStrategy` | string |  |  |  |
| `chunkingConfig` | object |  |  |  |
| `filterMetadata` | object |  |  |  |
| `ignoreLinks` | boolean | yes |  |  |


<a id="schema-siptransferrequestdto"></a>
#### `SipTransferRequestDto`

> Request to transfer a SIP participant to another destination

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `participantIdentity` | string | yes | minLength=`1` | The ID of the participant to transfer |
| `roomName` | string | yes | minLength=`1` | The current room name |
| `transferTo` | string | yes | minLength=`1` | The destination (phone number with tel: prefix or SIP URI) |
| `playDialtone` | boolean | yes |  | Play dial tone during transfer (if false, room audio plays) |


<a id="schema-sipurlresponsedto"></a>
#### `SipUrlResponseDto`

> SIP URL response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `sipUrl` | string | yes |  | SIP URL for telephony preset configuration |


<a id="schema-skillcreaterequest"></a>
#### `SkillCreateRequest`

> Request to create a new skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`2` ; maxLength=`64` ; pattern=`^[a-z0-9][a-z0-9-]*[a-z0-9]$` |  |
| `description` | string | yes | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |


<a id="schema-skillexecuterequest"></a>
#### `SkillExecuteRequest`

> Request to execute a skill script

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `scriptName` | string | yes | minLength=`1` |  |
| `args` | string[] |  |  |  |
| `timeout` | integer (int32) |  |  | Execution timeout in seconds (1–600) |


<a id="schema-skillexecuteresponse"></a>
#### `SkillExecuteResponse`

> Result of executing a skill script

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `skillId` | string | yes |  |  |
| `scriptName` | string | yes |  |  |
| `stdout` | string | yes |  |  |
| `stderr` | string | yes |  |  |
| `returncode` | integer (int32) | yes |  |  |


<a id="schema-skillfiledto"></a>
#### `SkillFileDto`

> A file belonging to a skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `path` | string | yes |  |  |
| `content` | string |  |  |  |


<a id="schema-skillresponse"></a>
#### `SkillResponse`

> Skill details

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `aiServiceSkillId` | string | yes |  |  |
| `name` | string | yes |  |  |
| `description` | string |  |  |  |
| `isActive` | boolean | yes |  |  |
| `organizationId` | integer (int64) | yes |  | Use organizationRefId instead |
| `organizationRefId` | string | yes |  |  |
| `projectId` | integer (int64) | yes |  | Use projectRefId instead |
| `projectRefId` | string | yes |  |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `updatedBy` | string | yes |  |  |


<a id="schema-skillupdaterequest"></a>
#### `SkillUpdateRequest`

> Request to partially update an existing skill

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `description` | string |  | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |
| `isActive` | boolean |  |  |  |


<a id="schema-skillv1createrequest"></a>
#### `SkillV1CreateRequest`

> Request to create a skill via the v1 API (includes projectId)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `projectRefId` | string | yes |  |  |
| `name` | string | yes | minLength=`2` ; maxLength=`64` ; pattern=`^[a-z0-9][a-z0-9-]*[a-z0-9]$` |  |
| `description` | string | yes | minLength=`0` ; maxLength=`1024` |  |
| `files` | [`SkillFileDto`](#schema-skillfiledto)[] |  |  |  |


<a id="schema-skillsconfiguration"></a>
#### `SkillsConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `skillIds` | string[] | yes |  | List of AI service skill UUIDs to load for this agent |


<a id="schema-sortobject"></a>
#### `SortObject`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `empty` | boolean |  |  |  |
| `sorted` | boolean |  |  |  |
| `unsorted` | boolean |  |  |  |


<a id="schema-spandto"></a>
#### `SpanDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `type` | string | yes |  |  |
| `duration` | string |  |  |  |
| `start_time` | string | yes |  |  |
| `end_time` | string | yes |  |  |
| `status` | string | yes |  |  |
| `input` | object |  |  |  |
| `output` | object |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-storageresourcecreaterequest"></a>
#### `StorageResourceCreateRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` | Name of the storage resource |
| `llmProviderRefId` | string |  |  | Reference ID (UUID) of the LLM provider configuration to use for embeddings. Obtain from GET /storage-resources/llm-providers (providerRefId field). Preferred over llmConfigId. |
| `llmConfigId` | integer (int64) |  |  | Internal numeric ID of the LLM provider configuration. Deprecated — use llmProviderRefId instead. |
| `embeddingModelId` | string | yes | minLength=`1` | ID of the embedding model to use |
| `metadataSchema` | [`MetadataSchemaField`](#schema-metadataschemafield)[] |  |  | Schema definition for metadata fields used to filter documents during search/retrieval |
| `invertedIndexConfig` | object |  |  | BM25 and inverted index tuning. bm25_b and bm25_k1 must be set together. See https://docs.weaviate.io/weaviate/config-refs/indexing/inverted-index |
| `vectorIndexType` | string |  | `hnsw`, `flat`, `dynamic`, `HNSW`, `FLAT`, `DYNAMIC` | Vector index type: hnsw (default), flat, or dynamic. |
| `vectorIndexConfig` | object |  |  | Vector index tuning. Available keys depend on vector_index_type (HNSW, FLAT, DYNAMIC). See https://docs.weaviate.io/weaviate/config-refs/indexing/vector-index |
| `searchConfig` | object |  |  | Search behavior configuration. Allows setting the search type (vector, keyword, or hybrid) and hybrid_search_alpha (0.0–1.0, must be 0.0 for keyword). |
| `isLlmProviderSpecified` | boolean | yes |  |  |


<a id="schema-storageresourceresponse"></a>
#### `StorageResourceResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `status` | string | yes |  |  |
| `files` | integer (int32) | yes |  |  |
| `size` | integer (int64) | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `crawling` | boolean | yes |  |  |
| `llmProviderRefId` | string (uuid) |  |  |  |
| `llmConfigId` | integer (int64) |  |  |  |
| `llmConfigName` | string |  |  |  |
| `embeddingModelId` | string | yes |  |  |
| `embeddingModelName` | string | yes |  |  |
| `embeddingModelDimensions` | integer (int32) |  |  |  |
| `metadataSchema` | [`MetadataSchemaField`](#schema-metadataschemafield)[] |  |  |  |
| `invertedIndexConfig` | object |  |  |  |
| `vectorIndexType` | string |  |  |  |
| `vectorIndexConfig` | object |  |  |  |
| `searchConfig` | object |  |  |  |


<a id="schema-storageresourceupdaterequest"></a>
#### `StorageResourceUpdateRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` |  |


<a id="schema-storageuploadconfigresponse"></a>
#### `StorageUploadConfigResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `maxFileSize` | integer (int64) | yes |  |  |
| `maxFileSizeMb` | integer (int32) | yes |  |  |
| `maxFilesUpload` | integer (int32) | yes |  |  |
| `supportedFileTypes` | object | yes |  |  |
| `timestamp` | string (date-time) | yes |  |  |


<a id="schema-structuredresponseconfig"></a>
#### `StructuredResponseConfig`

> Structured response configuration

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `structuredOutputs` | boolean |  |  | Enable structured outputs |
| `responseModel` | object |  |  | Response model schema |


<a id="schema-subdomainvalidationresponse"></a>
#### `SubdomainValidationResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `available` | boolean | yes |  |  |
| `reason` | string |  |  |  |


<a id="schema-ttsprovidertestrequest"></a>
#### `TTSProviderTestRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `provider` | string | yes | `azure`, `openai`, `cartesia`, `eleven` |  |
| `credentials` | object | yes |  |  |


<a id="schema-teamclonerequest"></a>
#### `TeamCloneRequest`

> Request to clone a team

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing name for the cloned team |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name for the cloned team shown in the platform dashboard |


<a id="schema-teamconfigurationlong"></a>
#### `TeamConfigurationLong`

> Team configuration settings

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `members` | [`TeamMemberConfiguration`](#schema-teammemberconfiguration)[] | yes |  | Team members (agents) |
| `model` | [`ModelConfigurationLong`](#schema-modelconfigurationlong) |  |  | Model configuration |
| `role` | string | yes |  | Team role |
| `instructions` | string |  |  | System instructions for the team |
| `advancedSettings` | [`AdvancedSettings`](#schema-advancedsettings) |  |  | Advanced settings for team behavior |
| `structuredResponseConfig` | [`StructuredResponseConfig`](#schema-structuredresponseconfig) |  |  | Structured response configuration |
| `addDatetimeToContext` | boolean |  |  | Add datetime to context (deprecated, use advancedSettings.addDatetimeToContext) |
| `addLocationToContext` | boolean |  |  | Add location to context (deprecated, use advancedSettings.addLocationToContext) |
| `reasoning` | boolean |  |  | Enable reasoning |
| `respondDirectly` | boolean |  |  | Respond directly without team leader processing |
| `delegateTaskToAllMembers` | boolean |  |  | Delegate task to all members simultaneously |
| `determineInputForMembers` | boolean |  |  | Team leader determines input for each member |
| `cacheSession` | boolean |  |  | Cache current team session |
| `addMemberToolsToContext` | boolean |  |  | Add member tools to system message context |
| `shareMemberInteractions` | boolean |  |  | Members can see and build upon each other's responses |
| `readTeamHistory` | boolean |  |  | Read team history during execution |
| `tools` | [`ToolConfiguration`](#schema-toolconfiguration)[] |  |  | List of tools enabled for this team |
| `mcp` | [`McpConfiguration`](#schema-mcpconfiguration)[] |  |  | MCP (Model Context Protocol) configurations |
| `httpRequests` | [`HttpRequestReference`](#schema-httprequestreference)[] |  |  | HTTP request references (minimal data, enriched before sending to AI service) |
| `memory` | [`MemoryConfiguration`](#schema-memoryconfiguration) |  |  | Memory configuration |
| `sessionStorage` | [`SessionStorageConfiguration`](#schema-sessionstorageconfiguration) |  |  | Session storage configuration |
| `agenticRag` | boolean |  |  | Enable Agentic RAG |
| `systemMetadataFiltering` | boolean |  |  | Enable automatic metadata filtering for knowledge base queries based on system context |
| `knowledgeBase` | [`KnowledgeBaseConfiguration`](#schema-knowledgebaseconfiguration)[] |  |  | Knowledge base configurations with storage resource UUIDs |
| `markdown` | boolean |  |  | Enable markdown formatting |
| `retryConfig` | [`AiRetryConfig`](#schema-airetryconfig) |  |  | Retry configuration |
| `toolExecutionSettings` | [`ToolExecutionSettings`](#schema-toolexecutionsettings) |  |  | Tool execution settings |
| `reasoningConfig` | object |  |  | Reasoning configuration |
| `mode` | string |  |  | Team mode: coordinate, route, broadcast, or tasks |
| `maxIterations` | integer (int32) |  | minimum=`1` ; maximum=`10` | Maximum iterations (1-10) |


<a id="schema-teamcreaterequest"></a>
#### `TeamCreateRequest`

> Request to create a new team

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name known to the AI (e.g. 'Support Team') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Support') |
| `projectId` | integer (int64) |  |  | ID of the project this team belongs to. Required when calling V1 API (/client/api/v1/teams). |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) | yes |  | Team configuration |


<a id="schema-teamexecutionresponse"></a>
#### `TeamExecutionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `run_id` | string | yes |  |  |
| `content` | object | yes |  |  |
| `session_id` | string | yes |  |  |
| `user_id` | string | yes |  |  |
| `messages` | object[] |  |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-teamjobcreateddto"></a>
#### `TeamJobCreatedDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `jobId` | string | yes |  |  |
| `teamId` | string | yes |  |  |
| `status` | string | yes | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `TIMEOUT`, `CANCELLED` |  |
| `userId` | string | yes |  |  |
| `sessionId` | string | yes |  |  |
| `message` | string | yes |  |  |


<a id="schema-teamjobresponsedto"></a>
#### `TeamJobResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `jobId` | string | yes |  |  |
| `teamId` | string | yes |  |  |
| `status` | string | yes | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `TIMEOUT`, `CANCELLED` |  |
| `userId` | string |  |  |  |
| `sessionId` | string |  |  |  |
| `result` | [`TeamJobResult`](#schema-teamjobresult) |  |  |  |
| `error` | string |  |  |  |
| `startedAt` | string (date-time) |  |  |  |
| `completedAt` | string (date-time) |  |  |  |


<a id="schema-teamjobresult"></a>
#### `TeamJobResult`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `runId` | string | yes |  |  |
| `content` | object | yes |  |  |
| `sessionId` | string | yes |  |  |
| `userId` | string | yes |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `messages` | object[] |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-teammemberconfiguration"></a>
#### `TeamMemberConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `memberType` | string | yes |  | Member type - must be 'agent' |
| `memberId` | string | yes |  | Agent ID - references agents.ai_service_agent_id |
| `memberName` | string |  |  | Member name |
| `memberRole` | string |  |  | Role description for this member within the team |


<a id="schema-teamoptionv1dto"></a>
#### `TeamOptionV1Dto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `aiServiceTeamId` | string | yes |  |  |


<a id="schema-teamresponse"></a>
#### `TeamResponse`

> Team information

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Internal database ID |
| `aiServiceTeamId` | string |  |  | AI service team ID |
| `name` | string | yes |  | LLM-facing team name known to the AI |
| `presetName` | string | yes |  | Display name shown in the platform dashboard |
| `description` | string |  |  | Team description |
| `organizationId` | integer (int64) | yes |  | Internal numeric organization ID. Deprecated — use organizationRefId. |
| `organizationRefId` | string | yes |  | Organization reference ID |
| `projectId` | integer (int64) | yes |  | Internal numeric project ID. Deprecated — use projectRefId. |
| `projectRefId` | string | yes |  | Project reference ID |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) | yes |  | Team configuration |
| `created` | string (date-time) | yes |  | Creation timestamp |
| `createdBy` | string | yes |  | Created by user |
| `updated` | string (date-time) | yes |  | Last update timestamp |
| `updatedBy` | string | yes |  | Updated by user |
| `status` | string |  |  | Team status from AI service |


<a id="schema-teamrunresponsedata"></a>
#### `TeamRunResponseData`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `run_id` | string | yes |  |  |
| `content` | object | yes |  |  |
| `session_id` | string | yes |  |  |
| `user_id` | string | yes |  |  |
| `metrics` | [`Metrics`](#schema-metrics) |  |  |  |
| `messages` | object[] |  |  |  |
| `metadata` | object |  |  |  |


<a id="schema-teamupdaterequest"></a>
#### `TeamUpdateRequest`

> Request to update an existing team (all fields optional)

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name (optional - only updates if provided) |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description (optional - only updates if provided) |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) |  |  | Team configuration (optional - only updates if provided) |


<a id="schema-teamv1createrequest"></a>
#### `TeamV1CreateRequest`

> Request to create a new team via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name known to the AI (e.g. 'Support Team') |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard (e.g. 'Dental Clinic Support') |
| `projectRefId` | string |  |  | Reference ID of the project this team belongs to (e.g. 'proj_...'). Use this instead of projectId. |
| `projectId` | integer (int64) |  |  | Numeric ID of the project. Deprecated — use projectRefId. |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) | yes |  | Team configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |


<a id="schema-teamv1updaterequest"></a>
#### `TeamV1UpdateRequest`

> Request to update an existing team via the v1 API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes | minLength=`1` ; maxLength=`100` | LLM-facing team name (optional - only updates if provided) |
| `presetName` | string | yes | minLength=`1` ; maxLength=`100` | Display name shown in the platform dashboard |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Team description (optional - only updates if provided) |
| `configuration` | [`TeamConfigurationLong`](#schema-teamconfigurationlong) |  |  | Team configuration. Set model.providerRefId to the provider UUID (preferred) and omit or set model.provider to 0. Or pass model.provider as a numeric ID (deprecated). |


<a id="schema-telephonypresetresponsedto"></a>
#### `TelephonyPresetResponseDto`

> Response DTO for telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Unique identifier |
| `organizationId` | integer (int64) | yes |  | Organization ID |
| `projectId` | integer (int64) | yes |  | Project ID |
| `name` | string | yes |  | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `referenceId` | string | yes |  | Reference ID for LiveKit integration |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |
| `created` | string (date-time) | yes |  | Creation timestamp |
| `createdBy` | string | yes |  | Creator username |
| `updated` | string (date-time) | yes |  | Last update timestamp |


<a id="schema-testconnectionresponse"></a>
#### `TestConnectionResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `status` | string | yes | `VALID`, `INVALID`, `NOT_VALIDATED` |  |
| `lastValidated` | string (date-time) |  |  |  |
| `error` | string |  |  |  |


<a id="schema-tokenresult"></a>
#### `TokenResult`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `accessToken` | string | yes |  |  |
| `refreshToken` | string | yes |  |  |
| `expirationTime` | integer (int64) | yes |  |  |


<a id="schema-toolconfiguration"></a>
#### `ToolConfiguration`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `toolId` | string | yes |  |  |
| `toolName` | string | yes |  |  |
| `category` | string |  |  |  |
| `enabled` | boolean | yes |  |  |
| `configuration` | object |  |  |  |
| `requiredPermissions` | string[] |  |  |  |


<a id="schema-toolexecutionsettings"></a>
#### `ToolExecutionSettings`

> Tool execution settings

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `toolCallLimit` | integer (int32) |  |  | Maximum tool calls per agent run |
| `maxToolCallsFromHistory` | integer (int32) |  |  | Maximum tool calls to include from chat history |
| `compressToolResults` | boolean |  |  | Whether to compress tool results to save context space |


<a id="schema-topapikeyusagedto"></a>
#### `TopApiKeyUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `totalRequests` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |


<a id="schema-topcomponentusagedto"></a>
#### `TopComponentUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `totalRequests` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |


<a id="schema-topprojectusagedto"></a>
#### `TopProjectUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `totalRequests` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |


<a id="schema-topuserusagedto"></a>
#### `TopUserUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `totalRequests` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |


<a id="schema-tracedetaildto"></a>
#### `TraceDetailDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `trace_id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `status` | string | yes |  |  |
| `duration` | string |  |  |  |
| `start_time` | string | yes |  |  |
| `end_time` | string |  |  |  |
| `total_spans` | integer (int32) | yes |  |  |
| `error_count` | integer (int32) | yes |  |  |
| `input` | object |  |  |  |
| `output` | object |  |  |  |
| `run_id` | string |  |  |  |
| `session_id` | string |  |  |  |
| `user_id` | string |  |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `workflow_id` | string |  |  |  |
| `created_at` | string | yes |  |  |
| `tree` | [`TraceNodeDto`](#schema-tracenodedto)[] |  |  |  |


<a id="schema-tracelistdto"></a>
#### `TraceListDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `trace_id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `status` | string | yes |  |  |
| `duration` | string |  |  |  |
| `start_time` | string | yes |  |  |
| `end_time` | string |  |  |  |
| `total_spans` | integer (int32) | yes |  |  |
| `error_count` | integer (int32) | yes |  |  |
| `run_id` | string |  |  |  |
| `session_id` | string |  |  |  |
| `user_id` | string |  |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `workflow_id` | string |  |  |  |
| `created_at` | string | yes |  |  |


<a id="schema-tracenodedto"></a>
#### `TraceNodeDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `type` | string | yes |  |  |
| `duration` | string |  |  |  |
| `start_time` | string | yes |  |  |
| `end_time` | string | yes |  |  |
| `status` | string | yes |  |  |
| `spans` | [`SpanDto`](#schema-spandto)[] |  |  |  |


<a id="schema-tracesessionstatsdto"></a>
#### `TraceSessionStatsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `session_id` | string | yes |  |  |
| `user_id` | string |  |  |  |
| `agent_id` | string |  |  |  |
| `team_id` | string |  |  |  |
| `workflow_id` | string |  |  |  |
| `total_traces` | integer (int32) | yes |  |  |
| `first_trace_at` | string | yes |  |  |
| `last_trace_at` | string | yes |  |  |


<a id="schema-transformconfig"></a>
#### `TransformConfig`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `pattern` | string | yes | minLength=`1` |  |
| `replacement` | string | yes | minLength=`1` |  |


<a id="schema-updatedocumentationrequest"></a>
#### `UpdateDocumentationRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `content` | string | yes |  |  |
| `contentType` | string | yes |  |  |


<a id="schema-updateevalrunrequest"></a>
#### `UpdateEvalRunRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  |  |  |
| `eval_data` | object |  |  |  |


<a id="schema-updatepasswordrequest"></a>
#### `UpdatePasswordRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `password` | string | yes |  |  |
| `newPassword` | string | yes |  |  |


<a id="schema-updateprofilerequest"></a>
#### `UpdateProfileRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `firstName` | string |  |  |  |
| `lastName` | string |  |  |  |


<a id="schema-updatesecretrequest"></a>
#### `UpdateSecretRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `key` | string |  | minLength=`0` ; maxLength=`255` |  |
| `value` | string |  |  |  |
| `description` | string |  | minLength=`0` ; maxLength=`1000` |  |
| `isEmpty` | boolean | yes |  |  |


<a id="schema-updatetelephonypresetdto"></a>
#### `UpdateTelephonyPresetDto`

> Request DTO for updating a telephony preset

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`100` | Name of the telephony preset |
| `type` | string | yes | `INBOUND`, `OUTBOUND` | Type of telephony preset |
| `inbound` | [`InboundConfigDto`](#schema-inboundconfigdto) | yes |  | Inbound telephony configuration |
| `outbound` | [`OutboundConfigDto`](#schema-outboundconfigdto) | yes |  | Outbound telephony configuration |


<a id="schema-updatewebhooksubscriptionrequest"></a>
#### `UpdateWebhookSubscriptionRequest`

> Request to update a webhook subscription

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string |  | minLength=`0` ; maxLength=`255` | Name of the webhook subscription |
| `endpointUrl` | string |  | minLength=`0` ; maxLength=`2048` ; pattern=`^https?://.*` | The webhook endpoint URL |
| `description` | string |  | minLength=`0` ; maxLength=`1000` | Description of the webhook subscription |
| `eventTypes` | string[] |  |  | List of event types to subscribe to |
| `maxDeliveryAttempts` | integer (int32) |  | minimum=`1` ; maximum=`10` | Maximum number of delivery attempts |
| `isActive` | boolean |  |  | Whether the subscription is active |


<a id="schema-urlchilddto"></a>
#### `UrlChildDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `url` | string | yes |  |  |
| `pageTitle` | string |  |  |  |
| `status` | string | yes |  |  |
| `error` | string |  |  |  |
| `hasFile` | boolean | yes |  |  |
| `fileSize` | integer (int64) |  |  |  |
| `type` | string | yes |  |  |
| `errorSource` | string |  |  |  |
| `filterMetadata` | object |  |  |  |


<a id="schema-usagedto"></a>
#### `UsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `from` | string (date-time) | yes |  |  |
| `to` | string (date-time) | yes |  |  |
| `totalRequests` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `totalInputTokens` | integer (int64) | yes |  |  |
| `totalOutputTokens` | integer (int64) | yes |  |  |
| `totalToolCalls` | integer (int64) | yes |  |  |
| `totalRagRetrievals` | integer (int64) | yes |  |  |
| `totalAudioDuration` | number | yes |  |  |
| `totalTtsChars` | integer (int64) | yes |  |  |
| `totalSttDuration` | number | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |


<a id="schema-usageresponsedto"></a>
#### `UsageResponseDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `granularity` | string | yes | `HOUR`, `DAY`, `WEEK`, `MONTH`, `YEAR` |  |
| `startDate` | string (date) | yes |  |  |
| `endDate` | string (date) | yes |  |  |
| `avgSuccessRate` | number | yes |  |  |
| `totalStorageBytes` | integer (int64) | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `data` | [`ProjectUsageDto`](#schema-projectusagedto)[] | yes |  |  |


<a id="schema-useractivationdto"></a>
#### `UserActivationDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `password` | string | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |


<a id="schema-userdto"></a>
#### `UserDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) |  |  |  |
| `referenceId` | string |  |  |  |
| `firstName` | string |  |  |  |
| `lastName` | string |  |  |  |
| `password` | string |  |  |  |
| `email` | string | yes |  |  |
| `role` | string | yes | `ROLE_ADMIN`, `ROLE_USER` |  |
| `enabled` | boolean | yes |  |  |
| `emailVerified` | boolean | yes |  |  |
| `mustChangePassword` | boolean | yes |  |  |
| `orgs` | [`OrganizationDto`](#schema-organizationdto)[] | yes |  |  |


<a id="schema-userusagedto"></a>
#### `UserUsageDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `email` | string | yes |  |  |
| `lastName` | string |  |  |  |
| `firstName` | string |  |  |  |
| `referenceId` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `updated` | string (date-time) | yes |  |  |
| `totalInputTokens` | integer (int64) | yes |  |  |
| `totalOutputTokens` | integer (int64) | yes |  |  |
| `totalTokens` | integer (int64) | yes |  |  |
| `avgSuccessRate` | number | yes |  |  |
| `avgExecutionDuration` | number | yes |  |  |
| `totalToolCalls` | integer (int64) | yes |  |  |
| `totalRagRetrievals` | integer (int64) | yes |  |  |
| `totalAudioDuration` | number | yes |  |  |
| `totalTtsChars` | integer (int64) | yes |  |  |
| `totalSttDuration` | number | yes |  |  |
| `totalDocumentPages` | integer (int64) | yes |  |  |
| `totalCreditsUsed` | integer (int64) | yes |  |  |
| `usage` | [`UsageDto`](#schema-usagedto)[] | yes |  |  |


<a id="schema-validatesubdomainrequest"></a>
#### `ValidateSubdomainRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `subdomain` | string | yes | minLength=`3` ; maxLength=`63` ; pattern=`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` |  |


<a id="schema-valueunitdto"></a>
#### `ValueUnitDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `value` | integer (int64) | yes |  |  |
| `unit` | string | yes |  |  |


<a id="schema-voice"></a>
#### `Voice`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | string | yes |  |  |
| `name` | string | yes |  |  |
| `gender` | string |  |  |  |
| `type` | string |  |  |  |
| `locale` | string |  |  |  |
| `labels` | string[] | yes |  |  |


<a id="schema-voicecredentialstest"></a>
#### `VoiceCredentialsTest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `status` | string | yes | `VALID`, `INVALID`, `UNKNOWN` |  |
| `lastValidated` | string (date-time) | yes |  |  |
| `error` | string |  |  |  |


<a id="schema-voicemoderesponse"></a>
#### `VoiceModeResponse`

> Voice mode configuration response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `mode` | string | yes | `LIVEKIT`, `WEBRTC` | Voice mode - LIVEKIT or WEBRTC |


<a id="schema-voicepresetclientdto"></a>
#### `VoicePresetClientDto`

> Voice preset information for discovery API

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Unique identifier of the audio agent |
| `name` | string | yes |  | Human-readable name of the audio agent |
| `voiceId` | string | yes |  | Voice ID used for WebRTC configuration |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  | Agent configuration settings |


<a id="schema-voicepresetdto"></a>
#### `VoicePresetDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `referenceId` | string | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `projectId` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-voicepresetoptiondto"></a>
#### `VoicePresetOptionDto`

> Voice preset option for dropdowns

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Voice preset ID |
| `referenceId` | string | yes |  | Voice preset reference ID |
| `name` | string | yes |  | Voice preset name |


<a id="schema-voicepresetrequestdto"></a>
#### `VoicePresetRequestDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `name` | string | yes |  |  |
| `settings` | [`VoicePresetSettingsDto`](#schema-voicepresetsettingsdto) | yes |  |  |


<a id="schema-voicepresetsettingsdto"></a>
#### `VoicePresetSettingsDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `llm_provider` | string | yes | `sirma` |  |
| `llm_config` | [`LlmConfig`](#schema-llmconfig) | yes |  |  |
| `tts_provider` | string | yes | `azure`, `openai`, `cartesia`, `eleven` |  |
| `tts_config` | object | yes |  |  |
| `stt_provider` | string | yes | `google`, `soniox`, `azure`, `cartesia`, `eleven`, `deepgram`, `assemblyai` |  |
| `stt_config` | object | yes |  |  |
| `avatar_provider` | string |  | `anam` |  |
| `avatar_config` | object |  |  |  |
| `turn_detection_provider` | string |  | `stt`, `vad`, `manual`, `realtime_llm`, `english_model`, `multilingual_model` |  |
| `turn_detection_config` | object |  |  |  |
| `vad_provider` | string |  | `silero`, `builtin` |  |
| `vad_config` | object |  |  |  |
| `interruption_config` | object |  |  |  |
| `call_behavior` | object |  |  |  |


<a id="schema-voicepresetsummarydto"></a>
#### `VoicePresetSummaryDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `organizationId` | integer (int64) | yes |  |  |
| `projectId` | integer (int64) | yes |  |  |
| `name` | string | yes |  |  |
| `created` | string (date-time) | yes |  |  |
| `createdBy` | string | yes |  |  |
| `updated` | string (date-time) | yes |  |  |


<a id="schema-voicepreviewresponse"></a>
#### `VoicePreviewResponse`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `previewUrl` | string | yes |  |  |


<a id="schema-voiceproviderdto"></a>
#### `VoiceProviderDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `key` | string | yes |  |  |
| `name` | string | yes |  |  |
| `models` | [`VoiceProviderModelDto`](#schema-voiceprovidermodeldto)[] | yes |  |  |


<a id="schema-voiceprovidermodeldto"></a>
#### `VoiceProviderModelDto`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  |  |
| `key` | string | yes |  |  |
| `name` | string | yes |  |  |


<a id="schema-voicerequest"></a>
#### `VoiceRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKey` | string | yes |  |  |


<a id="schema-voicesessionrequest"></a>
#### `VoiceSessionRequest`

> Optional Request to start a voice session

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `video` | [`VoiceVideoRequest`](#schema-voicevideorequest) | yes |  |  |


<a id="schema-voicesessionresponse"></a>
#### `VoiceSessionResponse`

> Response for starting a voice session

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `token` | string | yes |  | LiveKit access token for the user to join the room |
| `wsUrl` | string | yes |  | LiveKit WebSocket URL |
| `userId` | string | yes |  |  |
| `sessionId` | string | yes |  |  |


<a id="schema-voicevideorequest"></a>
#### `VoiceVideoRequest`

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `apiKey` | string | yes |  |  |
| `avatarId` | string | yes |  |  |
| `platform` | string | yes |  |  |


<a id="schema-webhookdeliveryresponse"></a>
#### `WebhookDeliveryResponse`

> Webhook delivery response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Delivery ID |
| `webhookId` | string | yes |  | Unique webhook ID |
| `sessionId` | string |  |  | Session ID associated with this webhook delivery (if applicable) |
| `eventType` | string | yes |  | Event type that was delivered |
| `status` | string | yes | `PENDING`, `DELIVERED`, `FAILED`, `PERMANENTLY_FAILED` | Current delivery status |
| `attemptCount` | integer (int32) | yes |  | Number of delivery attempts made |
| `maxAttempts` | integer (int32) | yes |  | Maximum number of attempts allowed |
| `nextRetryAt` | string (date-time) |  |  | When the next retry will be attempted |
| `lastAttemptAt` | string (date-time) |  |  | When the last delivery attempt was made |
| `lastHttpStatus` | integer (int32) |  |  | HTTP status code of the last attempt |
| `lastErrorMessage` | string |  |  | Error message from the last attempt |
| `responseTimeMs` | integer (int64) |  |  | Response time in milliseconds |
| `deliveredAt` | string (date-time) |  |  | When the webhook was successfully delivered |
| `subscription` | [`WebhookSubscriptionSummary`](#schema-webhooksubscriptionsummary) | yes |  | Subscription information |
| `created` | string (date-time) | yes |  | When the delivery was created |
| `payload` | object | yes |  | The webhook payload data |


<a id="schema-webhookdeliverystats"></a>
#### `WebhookDeliveryStats`

> Webhook delivery statistics

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `totalDeliveries` | integer (int64) | yes |  | Total number of deliveries |
| `successfulDeliveries` | integer (int64) | yes |  | Number of successful deliveries |
| `failedDeliveries` | integer (int64) | yes |  | Number of failed deliveries |
| `permanentlyFailedDeliveries` | integer (int64) | yes |  | Number of permanently failed deliveries |
| `pendingDeliveries` | integer (int64) | yes |  | Number of pending deliveries |
| `successRate` | number (double) | yes |  | Success rate as a percentage |


<a id="schema-webhookreplayrequest"></a>
#### `WebhookReplayRequest`

> Webhook replay request

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `force` | boolean | yes |  | Force replay even if delivery was successful |


<a id="schema-webhooksubscriptionresponse"></a>
#### `WebhookSubscriptionResponse`

> Webhook subscription response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Subscription ID |
| `name` | string | yes |  | Name of the webhook subscription |
| `endpointUrl` | string | yes |  | The webhook endpoint URL |
| `description` | string |  |  | Description of the webhook subscription |
| `eventTypes` | string[] | yes |  | List of event types subscribed to |
| `isActive` | boolean | yes |  | Whether the subscription is active |
| `maxDeliveryAttempts` | integer (int32) | yes |  | Maximum number of delivery attempts |
| `organization` | [`BaseIdNameDto`](#schema-baseidnamedto) | yes |  | Organization information |
| `project` | [`BaseIdNameDto`](#schema-baseidnamedto) | yes |  | Project information |
| `created` | string (date-time) | yes |  | When the subscription was created |
| `updated` | string (date-time) | yes |  | When the subscription was last updated |
| `secret` | string |  |  | Webhook signing secret (only returned on creation for security) |


<a id="schema-webhooksubscriptionsummary"></a>
#### `WebhookSubscriptionSummary`

> Webhook subscription summary

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `id` | integer (int64) | yes |  | Subscription ID |
| `name` | string | yes |  | Subscription name |
| `endpointUrl` | string | yes |  | Endpoint URL |


<a id="schema-webhooktestresponse"></a>
#### `WebhookTestResponse`

> Webhook test response

| Field | Type | Required | Constraints / Enum | Description |
|---|---|---|---|---|
| `webhookId` | string | yes |  | Test webhook ID |
| `httpStatus` | integer (int32) | yes |  | HTTP status code from the webhook endpoint |
| `responseTimeMs` | integer (int64) | yes |  | Response time in milliseconds |
| `success` | boolean | yes |  | Success status |
| `errorMessage` | string |  |  | Error message if the test failed |


---

## 13. Appendix: Full Endpoint Index

| Method | Path | Auth | Domain | Summary |
|---|---|---|---|---|
| `GET` | `/api/admin/organizations` | bearer-jwt | Organizations, Projects & Tenancy | Get all organizations |
| `POST` | `/api/admin/organizations` | bearer-jwt | Organizations, Projects & Tenancy | Create a new organization |
| `GET` | `/api/admin/organizations/{organizationId}/rate-limit` | bearer-jwt | Organizations, Projects & Tenancy | Get current rate limit configuration |
| `PUT` | `/api/admin/organizations/{organizationId}/rate-limit` | bearer-jwt | Organizations, Projects & Tenancy | Update rate limit configuration |
| `GET` | `/api/admin/organizations/{organizationId}/usage-summary` | bearer-jwt | Organizations, Projects & Tenancy | Get organization usage summary |
| `DELETE` | `/api/auth/account` | bearer-jwt | Authentication & Identity | Request account deletion |
| `GET` | `/api/auth/accountVerification` | public | Authentication & Identity | Verify user account |
| `PUT` | `/api/auth/changePassword` | public | Authentication & Identity | Change password with a token |
| `POST` | `/api/auth/generateResetPasswordLink` | public | Authentication & Identity | Generate a password reset link |
| `POST` | `/api/auth/google/callback` | public | Authentication & Identity | Exchange a Google authorization code for app JWT tokens (GSI redirect flow) |
| `POST` | `/api/auth/login` | public | Authentication & Identity | Login |
| `POST` | `/api/auth/logout` | bearer-jwt | Authentication & Identity | Logout the current user |
| `GET` | `/api/auth/me` | bearer-jwt | Authentication & Identity | Get the current user's information |
| `PUT` | `/api/auth/profile` | bearer-jwt | Authentication & Identity | Update the current user's profile |
| `POST` | `/api/auth/refreshAccessToken` | public | Authentication & Identity | Refresh the access token |
| `POST` | `/api/auth/signup` | public | Authentication & Identity | Register a new user |
| `PUT` | `/api/auth/updatePassword` | bearer-jwt | Authentication & Identity | Update the current user's password |
| `GET` | `/api/compliance-frameworks` | bearer-jwt | Governance: Policies & Compliance | List compliance frameworks |
| `GET` | `/api/config/storage-upload` | bearer-jwt | Platform, Config & Media | Get storage upload configuration |
| `GET` | `/api/documentation/sections` | bearer-token | Platform, Config & Media | Get all documentation sections |
| `GET` | `/api/documentation/sections/{sectionSlug}` | bearer-token | Platform, Config & Media | Get a specific documentation section with subsections |
| `GET` | `/api/documentation/sections/{sectionSlug}/{subsectionSlug}` | bearer-token | Platform, Config & Media | Get a specific documentation subsection |
| `PUT` | `/api/documentation/sections/{sectionSlug}/{subsectionSlug}` | bearer-token | Platform, Config & Media | Update a documentation subsection |
| `GET` | `/api/events` | bearer-jwt | Audio, Voice & Telephony |  |
| `GET` | `/api/events/test` | bearer-jwt | Audio, Voice & Telephony |  |
| `GET` | `/api/images/{documentId}/{filename}` | public | Platform, Config & Media | Get image |
| `POST` | `/api/invitations` | public | Authentication & Identity |  |
| `GET` | `/api/invitations/{token}` | public | Authentication & Identity |  |
| `PUT` | `/api/invitations/{token}` | public | Authentication & Identity |  |
| `GET` | `/api/languages` | bearer-jwt | Other |  |
| `GET` | `/api/llms` | bearer-jwt | Other |  |
| `GET` | `/api/mcp-tools` | bearer-jwt | Tools & External Integrations | List all available MCP tools |
| `POST` | `/api/mcp-tools/invoke` | bearer-jwt | Tools & External Integrations | Invoke MCP Tool |
| `GET` | `/api/media/{s3KeyEncoded}` | public | Platform, Config & Media | Get media file |
| `GET` | `/api/n8n-deployment/domain` | bearer-jwt | Tools & External Integrations | Get Route53 domain configuration |
| `GET` | `/api/n8n-deployment/ssh-public-key` | bearer-jwt | Tools & External Integrations | Get SSH public key for deployment |
| `POST` | `/api/organizations` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{orgId}/policies/statistics` | public | Governance: Policies & Compliance | Get policy statistics |
| `PUT` | `/api/organizations/{organizationId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/billing` | bearer-jwt | Billing |  |
| `PUT` | `/api/organizations/{organizationId}/billing/auto-recharge` | bearer-jwt | Billing |  |
| `POST` | `/api/organizations/{organizationId}/billing/balance/add` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/history` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/payment-methods` | bearer-jwt | Billing |  |
| `POST` | `/api/organizations/{organizationId}/billing/payment-methods` | bearer-jwt | Billing |  |
| `DELETE` | `/api/organizations/{organizationId}/billing/payment-methods/{paymentMethodId}` | bearer-jwt | Billing |  |
| `PUT` | `/api/organizations/{organizationId}/billing/payment-methods/{paymentMethodId}/default` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/preferences` | bearer-jwt | Billing |  |
| `PUT` | `/api/organizations/{organizationId}/billing/preferences` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/setup-intent` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/usage` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/usage/export` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/billing/usage/users` | bearer-jwt | Billing |  |
| `GET` | `/api/organizations/{organizationId}/keys` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/keys` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/keys/system` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/keys/system/regenerate` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `DELETE` | `/api/organizations/{organizationId}/keys/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/keys/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/keys/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/limits` | bearer-jwt | Organizations, Projects & Tenancy | Get organization limits |
| `GET` | `/api/organizations/{organizationId}/llm-providers` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/llm-providers` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/export` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/llm-providers/import` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/llm-providers/models` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/options` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/page` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/llm-providers/test-connection` | bearer-jwt | Other |  |
| `DELETE` | `/api/organizations/{organizationId}/llm-providers/{id}` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/{id}` | bearer-jwt | Other |  |
| `PUT` | `/api/organizations/{organizationId}/llm-providers/{id}` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/{id}/export` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/{id}/models` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/{id}/models/{modelId}/reasoning-params` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/llm-providers/{id}/test-connection` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llm-providers/{id}/usage` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/llms/providers` | bearer-jwt | Other | Get supported AI service providers |
| `DELETE` | `/api/organizations/{organizationId}/n8n-deployment` | bearer-jwt | Tools & External Integrations | Delete organization's n8n deployment |
| `GET` | `/api/organizations/{organizationId}/n8n-deployment` | bearer-jwt | Tools & External Integrations | Get organization's n8n deployment |
| `POST` | `/api/organizations/{organizationId}/n8n-deployment` | bearer-jwt | Tools & External Integrations | Initiate n8n deployment |
| `POST` | `/api/organizations/{organizationId}/n8n-deployment/cancel` | bearer-jwt | Tools & External Integrations | Cancel an in-progress deployment |
| `GET` | `/api/organizations/{organizationId}/n8n-deployment/logs` | bearer-jwt | Tools & External Integrations | Get deployment logs |
| `GET` | `/api/organizations/{organizationId}/n8n-deployment/status` | bearer-jwt | Tools & External Integrations | Get deployment status |
| `POST` | `/api/organizations/{organizationId}/n8n-deployment/validate-subdomain` | bearer-jwt | Tools & External Integrations | Validate subdomain availability |
| `GET` | `/api/organizations/{organizationId}/overview` | bearer-jwt | Organizations, Projects & Tenancy | Get organization overview |
| `GET` | `/api/organizations/{organizationId}/policies` | bearer-jwt | Governance: Policies & Compliance | List policies with context-specific enabled status |
| `POST` | `/api/organizations/{organizationId}/policies` | bearer-jwt | Governance: Policies & Compliance | Create organization policy |
| `GET` | `/api/organizations/{organizationId}/policies/status` | bearer-jwt | Governance: Policies & Compliance | Get effective enabled policies |
| `DELETE` | `/api/organizations/{organizationId}/policies/{policyId}` | bearer-jwt | Governance: Policies & Compliance | Delete organization policy |
| `PUT` | `/api/organizations/{organizationId}/policies/{policyId}` | bearer-jwt | Governance: Policies & Compliance | Update organization policy |
| `DELETE` | `/api/organizations/{organizationId}/policies/{policyId}/status` | bearer-jwt | Governance: Policies & Compliance | Disable policy at organization, project, or agent level |
| `PUT` | `/api/organizations/{organizationId}/policies/{policyId}/status` | bearer-jwt | Governance: Policies & Compliance | Enable policy at organization, project, or agent level |
| `GET` | `/api/organizations/{organizationId}/projects` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/projects` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/projects/options` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agent-creator` | bearer-jwt | Agents |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agent-creator/agent` | bearer-jwt | Agents |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agent-creator/create` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents` | bearer-jwt | Agents | List agents for a project |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents` | bearer-jwt | Agents | Create a new agent |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/llm-providers` | bearer-jwt | Agents | Get configured LLM providers for agents |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/options` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/providers` | bearer-jwt | Agents | Get supported AI service providers |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}` | bearer-jwt | Agents | Delete an agent |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}` | bearer-jwt | Agents | Get agent by ID |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}` | bearer-jwt | Agents | Update an agent |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/clone` | bearer-jwt | Agents | Clone an agent |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/memories` | bearer-jwt | Agents |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/memories/{memoryId}` | bearer-jwt | Agents |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run` | bearer-jwt | Agents | Execute agent (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/run/stream` | bearer-jwt | Agents | Execute agent (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/runs/{runId}/cancel` | bearer-jwt | Agents | Cancel an agent run |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/runs/{runId}/continue` | bearer-jwt | Agents | Continue a paused agent run (streaming) |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions` | bearer-jwt | Agents |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}` | bearer-jwt | Agents |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}/rename` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{agentId}/sessions/{sessionId}/runs` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/compare` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/{versionId}` | bearer-jwt | Agents |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/agents/{aiServiceAgentId}/versions/{versionId}/rollback` | bearer-jwt | Agents |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/mode` | bearer-jwt | Other | Get voice mode |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/options` | bearer-jwt | Other | Get voice preset options |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}` | bearer-jwt | Other |  |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}` | bearer-jwt | Other |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/start-session` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/stt/test-credentials` | bearer-jwt | Other | Test STT credentials |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/tts/test-credentials` | bearer-jwt | Other | Test TTS credentials |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/voices` | bearer-jwt | Other |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/audio-agents/{id}/voices/{voiceId}/preview` | bearer-jwt | Other | Get voice preview |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/eval-runs` | bearer-jwt | Observability: Traces, Evaluations & Usage | List evaluation runs |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/eval-runs` | bearer-jwt | Observability: Traces, Evaluations & Usage | Execute evaluation |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}` | bearer-jwt | Observability: Traces, Evaluations & Usage | Delete evaluation run |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}` | bearer-jwt | Observability: Traces, Evaluations & Usage | Get evaluation run |
| `PATCH` | `/api/organizations/{organizationId}/projects/{projectId}/eval-runs/{evalRunId}` | bearer-jwt | Observability: Traces, Evaluations & Usage | Update evaluation run |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests` | bearer-jwt | Tools & External Integrations | List HTTP requests |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests` | bearer-jwt | Tools & External Integrations | Create HTTP request |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests/test` | bearer-jwt | Tools & External Integrations | Test HTTP request |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}` | bearer-jwt | Tools & External Integrations | Delete HTTP request |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}` | bearer-jwt | Tools & External Integrations | Get HTTP request |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/http-requests/{id}` | bearer-jwt | Tools & External Integrations | Update HTTP request |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/logs` | bearer-jwt | Agents | Get agent runs |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/logs/{runId}` | bearer-jwt | Agents | Get agent runs details |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers` | bearer-jwt | Tools & External Integrations | List MCP servers for a project |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers` | bearer-jwt | Tools & External Integrations | Create a new MCP server |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}` | bearer-jwt | Tools & External Integrations | Delete an MCP server |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}` | bearer-jwt | Tools & External Integrations | Get MCP server by UUID |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}` | bearer-jwt | Tools & External Integrations | Update an MCP server |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/health` | bearer-jwt | Tools & External Integrations | Get MCP server health status |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/test` | bearer-jwt | Tools & External Integrations | Test MCP server |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/tools` | bearer-jwt | Tools & External Integrations | List available tools for an MCP server |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/mcp-servers/{serverId}/tools/{toolName}/call` | bearer-jwt | Tools & External Integrations | Call a specific tool on an MCP server |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/run-stream` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/runs` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agent-creator/sessions/{sessionId}/runs` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agents/{agentId}/run-stream` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/agents/{agentId}/runs` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/playground/config/limits` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/teams/{teamId}/run-stream` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/teams/{teamId}/runs` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/workflows/{workflowId}/run-stream` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/playground/workflows/{workflowId}/runs` | bearer-jwt | Observability: Traces, Evaluations & Usage |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/skills` | bearer-jwt | Tools & External Integrations | List skills for a project |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/skills` | bearer-jwt | Tools & External Integrations | Create a new skill |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}` | bearer-jwt | Tools & External Integrations | Soft-delete a skill |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}` | bearer-jwt | Tools & External Integrations | Get a skill by ID |
| `PATCH` | `/api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}` | bearer-jwt | Tools & External Integrations | Partially update a skill |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}/activate` | bearer-jwt | Tools & External Integrations | Reactivate a soft-deleted skill |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/skills/{skillId}/execute` | bearer-jwt | Tools & External Integrations | Execute a skill script in a sandbox |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources` | bearer-jwt | Knowledge, Storage & Embeddings | List storage resources |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources` | bearer-jwt | Knowledge, Storage & Embeddings | Create storage resource |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/chunking-strategies` | bearer-jwt | Knowledge, Storage & Embeddings | Get chunking strategies |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/llm-providers` | bearer-jwt | Knowledge, Storage & Embeddings | Get configured embedding model providers |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}` | bearer-jwt | Knowledge, Storage & Embeddings | Delete storage resource |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}` | bearer-jwt | Knowledge, Storage & Embeddings | Get storage resource |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{id}` | bearer-jwt | Knowledge, Storage & Embeddings | Update storage resource |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/crawl` | bearer-jwt | Knowledge, Storage & Embeddings | Crawl URLs |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files` | bearer-jwt | Knowledge, Storage & Embeddings | List files |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files` | bearer-jwt | Knowledge, Storage & Embeddings | Upload files to storage |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}` | bearer-jwt | Knowledge, Storage & Embeddings | Delete file |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}/download` | bearer-jwt | Knowledge, Storage & Embeddings | Download file |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/files/{fileId}/parsed-text/download` | bearer-jwt | Knowledge, Storage & Embeddings | Download parsed text |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/storage-resources/{vectorStoreId}/search` | bearer-jwt | Knowledge, Storage & Embeddings | Search vector store |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams` | bearer-jwt | Teams | List teams for a project |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams` | bearer-jwt | Teams | Create a new team |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/options` | bearer-jwt | Teams |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}` | bearer-jwt | Teams | Delete a team |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}` | bearer-jwt | Teams | Get team by ID |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}` | bearer-jwt | Teams | Update a team |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/clone` | bearer-jwt | Teams | Clone a team |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/memories` | bearer-jwt | Teams |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/memories/{memoryId}` | bearer-jwt | Teams |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run` | bearer-jwt | Teams | Execute team (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/run/stream` | bearer-jwt | Teams | Execute team (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/runs/{runId}/cancel` | bearer-jwt | Teams | Cancel a team run |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions` | bearer-jwt | Teams |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}` | bearer-jwt | Teams |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}` | bearer-jwt | Teams |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}/rename` | bearer-jwt | Teams |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/teams/{teamId}/sessions/{sessionId}/runs` | bearer-jwt | Teams |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets` | public | Audio, Voice & Telephony | List telephony presets |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets` | public | Audio, Voice & Telephony | Create telephony preset |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}` | public | Audio, Voice & Telephony | Delete telephony preset |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}` | public | Audio, Voice & Telephony | Get telephony preset |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}` | public | Audio, Voice & Telephony | Update telephony preset |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/telephony-presets/{id}/outbound-call` | public | Audio, Voice & Telephony | Create outbound call |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/traces` | bearer-jwt | Observability: Traces, Evaluations & Usage | List traces |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/traces/session-stats` | bearer-jwt | Observability: Traces, Evaluations & Usage | Get trace session statistics |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/traces/{traceId}` | bearer-jwt | Observability: Traces, Evaluations & Usage | Get trace detail |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/users` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/users` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/users/options` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/users/{userId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/projects/{projectId}/users/{userId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/users/{userId}/resend-invitation` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/workflows` | bearer-jwt | Workflows | List available workflows |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}` | bearer-jwt | Workflows | Get workflow details |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run` | bearer-jwt | Workflows | Execute workflow (non-streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/run/stream` | bearer-jwt | Workflows | Execute workflow (streaming) |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/runs/{runId}/cancel` | bearer-jwt | Workflows | Cancel a workflow run |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions` | bearer-jwt | Workflows |  |
| `DELETE` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}` | bearer-jwt | Workflows |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}` | bearer-jwt | Workflows |  |
| `POST` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}/rename` | bearer-jwt | Workflows |  |
| `GET` | `/api/organizations/{organizationId}/projects/{projectId}/workflows/{workflowId}/sessions/{sessionId}/runs` | bearer-jwt | Workflows |  |
| `GET` | `/api/organizations/{organizationId}/role-presets` | bearer-jwt | Organizations, Projects & Tenancy | List all role presets |
| `POST` | `/api/organizations/{organizationId}/role-presets` | bearer-jwt | Organizations, Projects & Tenancy | Create a new role preset |
| `DELETE` | `/api/organizations/{organizationId}/role-presets/{presetId}` | bearer-jwt | Organizations, Projects & Tenancy | Delete a role preset |
| `GET` | `/api/organizations/{organizationId}/role-presets/{presetId}` | bearer-jwt | Organizations, Projects & Tenancy | Get a role preset by ID |
| `PUT` | `/api/organizations/{organizationId}/role-presets/{presetId}` | bearer-jwt | Organizations, Projects & Tenancy | Update a role preset |
| `GET` | `/api/organizations/{organizationId}/secrets` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/secrets` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `DELETE` | `/api/organizations/{organizationId}/secrets/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/secrets/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PATCH` | `/api/organizations/{organizationId}/secrets/{id}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/setup` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `GET` | `/api/organizations/{organizationId}/tags` | bearer-jwt | Organizations, Projects & Tenancy | Get organization tags |
| `GET` | `/api/organizations/{organizationId}/users` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/users` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `DELETE` | `/api/organizations/{organizationId}/users/{userId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `PUT` | `/api/organizations/{organizationId}/users/{userId}` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/organizations/{organizationId}/users/{userId}/resend-invitation` | bearer-jwt | Organizations, Projects & Tenancy |  |
| `POST` | `/api/policies` | bearer-jwt | Governance: Policies & Compliance | Create default policy |
| `DELETE` | `/api/policies/{id}` | bearer-jwt | Governance: Policies & Compliance | Delete default policy |
| `PUT` | `/api/policies/{id}` | bearer-jwt | Governance: Policies & Compliance | Update default policy |
| `POST` | `/api/policy-test` | bearer-jwt | Governance: Policies & Compliance | Test policy configuration |
| `GET` | `/api/public/config` | public | Platform, Config & Media | Get application configuration |
| `GET` | `/api/public/sip-url` | public | Platform, Config & Media | Get SIP URL |
| `PUT` | `/api/service/agents/{aiServiceAgentId}` | service-key-agent-creator/service-name | Agents |  |
| `GET` | `/api/service/agents/{aiServiceAgentId}/http-requests` | service-key-agent-creator/service-name | Agents |  |
| `GET` | `/api/service/agents/{aiServiceAgentId}/llm-providers` | service-key-agent-creator/service-name | Agents |  |
| `GET` | `/api/service/agents/{aiServiceAgentId}/mcp-servers` | service-key-agent-creator/service-name | Agents |  |
| `GET` | `/api/service/agents/{aiServiceAgentId}/storage-resources` | service-key-agent-creator/service-name | Agents |  |
| `POST` | `/api/service/audio/audio-session/notification` | service-key-audio/service-name | Audio, Voice & Telephony |  |
| `POST` | `/api/service/audio/inbound-session` | service-key-audio/service-name | Audio, Voice & Telephony |  |
| `DELETE` | `/api/service/audio/rooms/{roomName}` | service-key-audio/service-name | Audio, Voice & Telephony |  |
| `POST` | `/api/service/audio/sip/transfer` | service-key-audio/service-name | Audio, Voice & Telephony |  |
| `GET` | `/api/v1/organizations/{orgId}/policy-audit-logs` | public | Governance: Policies & Compliance | List policy audit logs |
| `GET` | `/api/v1/organizations/{orgId}/policy-audit-logs/export` | public | Governance: Policies & Compliance | Export policy audit logs |
| `GET` | `/api/v1/organizations/{orgId}/policy-audit-logs/statistics` | public | Governance: Policies & Compliance | Get audit log statistics |
| `GET` | `/api/vault/health` | public | Other |  |
| `GET` | `/api/voice-providers` | bearer-jwt | Other |  |
| `DELETE` | `/api/voice-providers/cache/clear` | bearer-jwt | Other |  |
| `POST` | `/api/voice-providers/stt/test-credentials` | bearer-jwt | Other |  |
| `POST` | `/api/voice-providers/tts/test-credentials` | bearer-jwt | Other |  |
| `GET` | `/api/voice-providers/{providerId}/models` | bearer-jwt | Other |  |
| `POST` | `/api/voice-providers/{providerId}/voices` | bearer-jwt | Other |  |
| `POST` | `/api/voice-providers/{providerId}/voices/{voiceId}/preview` | bearer-jwt | Other |  |
| `POST` | `/api/webhook/stripe` | public | Webhooks & Eventing |  |
| `GET` | `/api/webhooks/deliveries` | bearer-token | Webhooks & Eventing | List webhook deliveries |
| `POST` | `/api/webhooks/deliveries/{id}/replay` | bearer-token | Webhooks & Eventing | Replay webhook delivery |
| `GET` | `/api/webhooks/event-types` | bearer-token | Webhooks & Eventing | List supported event types |
| `GET` | `/api/webhooks/subscriptions` | bearer-token | Webhooks & Eventing | List webhook subscriptions |
| `POST` | `/api/webhooks/subscriptions` | bearer-token | Webhooks & Eventing | Create webhook subscription |
| `DELETE` | `/api/webhooks/subscriptions/{id}` | bearer-token | Webhooks & Eventing | Delete webhook subscription |
| `GET` | `/api/webhooks/subscriptions/{id}` | bearer-token | Webhooks & Eventing | Get webhook subscription |
| `PUT` | `/api/webhooks/subscriptions/{id}` | bearer-token | Webhooks & Eventing | Update webhook subscription |
| `POST` | `/api/webhooks/subscriptions/{id}/test` | bearer-token | Webhooks & Eventing | Test webhook endpoint |
| `GET` | `/api/webhooks/subscriptions/{subscriptionId}/statistics` | bearer-token | Webhooks & Eventing | Get webhook delivery statistics |
| `GET` | `/client/api/v1/agents` | api-key | Platform, Config & Media | List available agents |
| `POST` | `/client/api/v1/agents` | api-key | Platform, Config & Media | Create a new agent |
| `GET` | `/client/api/v1/agents/jobs/{jobId}/status` | api-key | Platform, Config & Media | Get agent job status |
| `GET` | `/client/api/v1/agents/llm-providers` | api-key | Platform, Config & Media | Get configured LLM providers for agents |
| `GET` | `/client/api/v1/agents/options` | api-key | Platform, Config & Media |  |
| `DELETE` | `/client/api/v1/agents/{agentId}` | api-key | Platform, Config & Media | Delete an agent |
| `GET` | `/client/api/v1/agents/{agentId}` | api-key | Platform, Config & Media | Get agent by ID |
| `PUT` | `/client/api/v1/agents/{agentId}` | api-key | Platform, Config & Media | Update an agent |
| `POST` | `/client/api/v1/agents/{agentId}/clone` | api-key | Platform, Config & Media | Clone an agent |
| `POST` | `/client/api/v1/agents/{agentId}/run` | api-key | Platform, Config & Media | Execute agent synchronously |
| `POST` | `/client/api/v1/agents/{agentId}/run-async` | api-key | Platform, Config & Media | Execute agent asynchronously |
| `POST` | `/client/api/v1/agents/{agentId}/run-stream` | api-key | Platform, Config & Media | Execute agent with streaming response |
| `POST` | `/client/api/v1/agents/{agentId}/runs/{runId}/cancel` | api-key | Platform, Config & Media | Cancel an agent run |
| `POST` | `/client/api/v1/agents/{agentId}/runs/{runId}/continue` | api-key | Platform, Config & Media | Continue an agent run with streaming response |
| `GET` | `/client/api/v1/agents/{agentId}/sessions` | api-key | Agents | List agent sessions |
| `DELETE` | `/client/api/v1/agents/{agentId}/sessions/{sessionId}` | api-key | Agents | Delete agent session |
| `GET` | `/client/api/v1/agents/{agentId}/sessions/{sessionId}` | api-key | Agents | Get agent session by ID |
| `POST` | `/client/api/v1/agents/{agentId}/sessions/{sessionId}/rename` | api-key | Agents | Rename agent session |
| `GET` | `/client/api/v1/agents/{agentId}/sessions/{sessionId}/runs` | api-key | Agents | Get agent session runs |
| `GET` | `/client/api/v1/audio-agents` | api-key | Audio, Voice & Telephony | List available audio agents |
| `POST` | `/client/api/v1/audio-agents` | api-key | Audio, Voice & Telephony | Create a new audio agent |
| `GET` | `/client/api/v1/audio-agents/mode` | api-key | Audio, Voice & Telephony | Get voice mode |
| `GET` | `/client/api/v1/audio-agents/options` | api-key | Audio, Voice & Telephony | Get voice preset options |
| `DELETE` | `/client/api/v1/audio-agents/{idOrRef}` | api-key | Audio, Voice & Telephony | Delete an audio agent |
| `GET` | `/client/api/v1/audio-agents/{idOrRef}` | api-key | Audio, Voice & Telephony | Get audio agent |
| `PUT` | `/client/api/v1/audio-agents/{idOrRef}` | api-key | Audio, Voice & Telephony | Update an audio agent |
| `POST` | `/client/api/v1/audio-agents/{idOrRef}/start-session` | api-key | Audio, Voice & Telephony | Start voice session |
| `GET` | `/client/api/v1/audio-agents/{idOrRef}/stt/test-credentials` | api-key | Audio, Voice & Telephony | Test STT credentials |
| `GET` | `/client/api/v1/audio-agents/{idOrRef}/tts/test-credentials` | api-key | Audio, Voice & Telephony | Test TTS credentials |
| `GET` | `/client/api/v1/audio-agents/{idOrRef}/voices` | api-key | Audio, Voice & Telephony | Get available voices for audio agent |
| `GET` | `/client/api/v1/audio-agents/{idOrRef}/voices/{voiceId}/preview` | api-key | Audio, Voice & Telephony | Get voice preview |
| `POST` | `/client/api/v1/embeddings` | api-key | Knowledge, Storage & Embeddings | Generate embeddings (OpenAI-compatible proxy) |
| `GET` | `/client/api/v1/eval-runs` | api-key | Platform, Config & Media | List evaluation runs |
| `POST` | `/client/api/v1/eval-runs` | api-key | Platform, Config & Media | Execute evaluation |
| `DELETE` | `/client/api/v1/eval-runs/{evalRunId}` | api-key | Platform, Config & Media | Delete evaluation run |
| `GET` | `/client/api/v1/eval-runs/{evalRunId}` | api-key | Platform, Config & Media | Get evaluation run |
| `PATCH` | `/client/api/v1/eval-runs/{evalRunId}` | api-key | Platform, Config & Media | Update evaluation run |
| `GET` | `/client/api/v1/health` | api-key | Platform, Config & Media | Check API health status |
| `GET` | `/client/api/v1/http-requests` | api-key | Platform, Config & Media | List HTTP requests |
| `POST` | `/client/api/v1/http-requests` | api-key | Platform, Config & Media | Create HTTP request |
| `POST` | `/client/api/v1/http-requests/test` | api-key | Platform, Config & Media | Test HTTP request |
| `DELETE` | `/client/api/v1/http-requests/{idOrRef}` | api-key | Platform, Config & Media | Delete HTTP request |
| `GET` | `/client/api/v1/http-requests/{idOrRef}` | api-key | Platform, Config & Media | Get HTTP request |
| `PUT` | `/client/api/v1/http-requests/{idOrRef}` | api-key | Platform, Config & Media | Update HTTP request |
| `GET` | `/client/api/v1/logs` | api-key | Platform, Config & Media | Get agent run logs |
| `GET` | `/client/api/v1/logs/by-run-id/{runId}` | api-key | Platform, Config & Media | Get agent run log details by run ID |
| `GET` | `/client/api/v1/logs/{id}` | api-key | Platform, Config & Media | Get agent run log details by numeric ID (deprecated) |
| `GET` | `/client/api/v1/organizations/limits` | api-key | Platform, Config & Media | Get organization limits |
| `GET` | `/client/api/v1/projects` | api-key | Organizations, Projects & Tenancy | List projects |
| `POST` | `/client/api/v1/projects` | api-key | Organizations, Projects & Tenancy | Create project |
| `DELETE` | `/client/api/v1/projects/{projectRefId}` | api-key | Organizations, Projects & Tenancy | Archive project |
| `GET` | `/client/api/v1/projects/{projectRefId}` | api-key | Organizations, Projects & Tenancy | Get project |
| `PUT` | `/client/api/v1/projects/{projectRefId}` | api-key | Organizations, Projects & Tenancy | Update project |
| `GET` | `/client/api/v1/projects/{projectRefId}/keys` | api-key | Platform, Config & Media | List API keys |
| `POST` | `/client/api/v1/projects/{projectRefId}/keys` | api-key | Platform, Config & Media | Create API key |
| `DELETE` | `/client/api/v1/projects/{projectRefId}/keys/{keyRefId}` | api-key | Platform, Config & Media | Revoke API key |
| `GET` | `/client/api/v1/projects/{projectRefId}/keys/{keyRefId}` | api-key | Platform, Config & Media | Get API key |
| `PUT` | `/client/api/v1/projects/{projectRefId}/keys/{keyRefId}` | api-key | Platform, Config & Media | Update API key |
| `GET` | `/client/api/v1/skills` | api-key | Platform, Config & Media | List skills |
| `POST` | `/client/api/v1/skills` | api-key | Platform, Config & Media | Create a skill |
| `DELETE` | `/client/api/v1/skills/{skillId}` | api-key | Platform, Config & Media | Soft-delete a skill |
| `GET` | `/client/api/v1/skills/{skillId}` | api-key | Platform, Config & Media | Get skill by ID |
| `PATCH` | `/client/api/v1/skills/{skillId}` | api-key | Platform, Config & Media | Update a skill |
| `POST` | `/client/api/v1/skills/{skillId}/activate` | api-key | Platform, Config & Media | Reactivate a soft-deleted skill |
| `POST` | `/client/api/v1/skills/{skillId}/execute` | api-key | Platform, Config & Media | Execute a skill script in a sandbox |
| `GET` | `/client/api/v1/storage-resources` | api-key | Platform, Config & Media | List storage resources |
| `POST` | `/client/api/v1/storage-resources` | api-key | Platform, Config & Media | Create a new storage resource |
| `GET` | `/client/api/v1/storage-resources/llm-providers` | api-key | Platform, Config & Media | Get configured LLM providers with embedding models |
| `DELETE` | `/client/api/v1/storage-resources/{vectorStoreId}` | api-key | Platform, Config & Media | Delete a storage resource |
| `GET` | `/client/api/v1/storage-resources/{vectorStoreId}` | api-key | Platform, Config & Media | Get storage resource by ID |
| `PUT` | `/client/api/v1/storage-resources/{vectorStoreId}` | api-key | Platform, Config & Media | Update storage resource |
| `POST` | `/client/api/v1/storage-resources/{vectorStoreId}/crawl` | api-key | Platform, Config & Media | Crawl URLs |
| `GET` | `/client/api/v1/storage-resources/{vectorStoreId}/files` | api-key | Platform, Config & Media | List files in storage resource |
| `POST` | `/client/api/v1/storage-resources/{vectorStoreId}/files` | api-key | Platform, Config & Media | Upload files to storage resource |
| `DELETE` | `/client/api/v1/storage-resources/{vectorStoreId}/files/{fileId}` | api-key | Platform, Config & Media | Delete a file from storage resource |
| `GET` | `/client/api/v1/storage-resources/{vectorStoreId}/files/{fileId}/download` | api-key | Platform, Config & Media | Download file |
| `GET` | `/client/api/v1/tags` | api-key | Platform, Config & Media | Get organization tags |
| `GET` | `/client/api/v1/teams` | api-key | Platform, Config & Media | List available teams |
| `POST` | `/client/api/v1/teams` | api-key | Platform, Config & Media | Create a new team |
| `GET` | `/client/api/v1/teams/jobs/{jobId}/status` | api-key | Platform, Config & Media | Get team job status |
| `GET` | `/client/api/v1/teams/options` | api-key | Platform, Config & Media |  |
| `DELETE` | `/client/api/v1/teams/{teamId}` | api-key | Platform, Config & Media | Delete a team |
| `GET` | `/client/api/v1/teams/{teamId}` | api-key | Platform, Config & Media | Get team by ID |
| `PUT` | `/client/api/v1/teams/{teamId}` | api-key | Platform, Config & Media | Update a team |
| `POST` | `/client/api/v1/teams/{teamId}/clone` | api-key | Platform, Config & Media | Clone a team |
| `POST` | `/client/api/v1/teams/{teamId}/run` | api-key | Platform, Config & Media | Execute team (non-streaming) |
| `POST` | `/client/api/v1/teams/{teamId}/run-async` | api-key | Platform, Config & Media | Execute team asynchronously |
| `POST` | `/client/api/v1/teams/{teamId}/run-stream` | api-key | Platform, Config & Media | Execute team (streaming) |
| `POST` | `/client/api/v1/teams/{teamId}/runs/{runId}/cancel` | api-key | Platform, Config & Media | Cancel a team run |
| `GET` | `/client/api/v1/teams/{teamId}/sessions` | api-key | Platform, Config & Media | List team sessions |
| `DELETE` | `/client/api/v1/teams/{teamId}/sessions/{sessionId}` | api-key | Platform, Config & Media | Delete team session |
| `GET` | `/client/api/v1/teams/{teamId}/sessions/{sessionId}` | api-key | Platform, Config & Media | Get team session by ID |
| `POST` | `/client/api/v1/teams/{teamId}/sessions/{sessionId}/rename` | api-key | Platform, Config & Media | Rename team session |
| `GET` | `/client/api/v1/teams/{teamId}/sessions/{sessionId}/runs` | api-key | Platform, Config & Media | Get team session runs |
| `GET` | `/client/api/v1/telephony-presets` | api-key | Platform, Config & Media | List telephony presets |
| `POST` | `/client/api/v1/telephony-presets` | api-key | Platform, Config & Media | Create telephony preset |
| `DELETE` | `/client/api/v1/telephony-presets/{idOrRef}` | api-key | Platform, Config & Media | Delete telephony preset |
| `GET` | `/client/api/v1/telephony-presets/{idOrRef}` | api-key | Platform, Config & Media | Get telephony preset |
| `PUT` | `/client/api/v1/telephony-presets/{idOrRef}` | api-key | Platform, Config & Media | Update telephony preset |
| `POST` | `/client/api/v1/telephony-presets/{idOrRef}/outbound-call` | api-key | Platform, Config & Media | Create outbound call |
| `GET` | `/client/api/v1/traces` | api-key | Platform, Config & Media | List traces |
| `GET` | `/client/api/v1/traces/session-stats` | api-key | Platform, Config & Media | Get trace session statistics |
| `GET` | `/client/api/v1/traces/{traceId}` | api-key | Platform, Config & Media | Get trace detail |
| `GET` | `/client/api/v1/usage` | api-key | Platform, Config & Media |  |
| `GET` | `/client/api/v1/usage/export` | api-key | Platform, Config & Media |  |
| `GET` | `/client/api/v1/version` | api-key | Platform, Config & Media | Get API version information |
| `GET` | `/client/api/v1/workflows` | api-key | Platform, Config & Media | List available workflows |
| `GET` | `/client/api/v1/workflows/{workflowId}` | api-key | Platform, Config & Media | Get workflow details |
| `POST` | `/client/api/v1/workflows/{workflowId}/run` | api-key | Platform, Config & Media | Execute workflow synchronously |
| `POST` | `/client/api/v1/workflows/{workflowId}/run-stream` | api-key | Platform, Config & Media | Execute workflow with streaming response |
| `POST` | `/client/api/v1/workflows/{workflowId}/runs/{runId}/cancel` | api-key | Platform, Config & Media | Cancel a workflow run |
| `GET` | `/client/api/v1/workflows/{workflowId}/sessions` | api-key | Workflows | List workflow sessions |
| `DELETE` | `/client/api/v1/workflows/{workflowId}/sessions/{sessionId}` | api-key | Workflows | Delete workflow session |
| `GET` | `/client/api/v1/workflows/{workflowId}/sessions/{sessionId}` | api-key | Workflows | Get workflow session by ID |
| `POST` | `/client/api/v1/workflows/{workflowId}/sessions/{sessionId}/rename` | api-key | Workflows | Rename workflow session |
| `GET` | `/client/api/v1/workflows/{workflowId}/sessions/{sessionId}/runs` | api-key | Workflows | Get workflow session runs |
| `GET` | `/health` | public | Other |  |


---

_End of guide. Generated from `api-docs_AgenticSAI_.json` on 2026-05-17. 404 operations, 364 schemas._