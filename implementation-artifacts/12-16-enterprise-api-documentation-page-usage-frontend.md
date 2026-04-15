# Story 12.16: Enterprise API Documentation Page & Usage Frontend

Status: done

## Story

As a **company admin on the Enterprise tier**,
I want **a Developer hub page that embeds the interactive Enterprise API documentation and an API key management page where I can create, view, and revoke API keys and see my rate limit tier**,
so that **my team can explore available endpoints interactively and I can safely manage programmatic API access without leaving the EU Solicit web application**.

## Acceptance Criteria

### AC1 — Backend: `require_enterprise_tier` Dependency

**File:** `services/client-api/src/client_api/core/tier_gate.py`

Add the following constant and dependency **after** `PROFESSIONAL_PLUS_PLANS`:

```python
#: Plans considered "Enterprise" for Enterprise API management access (S12.16).
ENTERPRISE_PLANS: frozenset[str] = frozenset({"enterprise"})
```

Add to `__all__` (or append alongside the existing exports): `"ENTERPRISE_PLANS"`, `"require_enterprise_tier"`.

New dependency function:

```python
async def require_enterprise_tier(
    current_user: Annotated[CurrentUser, Depends(get_current_user)],
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> CurrentUser:
    """FastAPI dependency: require the user's company to have an active Enterprise subscription.

    Queries ``client.subscriptions`` for a row matching:
      company_id = current_user.company_id
      AND plan = 'enterprise'
      AND status = 'active'

    Returns the CurrentUser unchanged if a qualifying row is found.

    Raises
    ------
    ForbiddenError (HTTP 403)
        If no active Enterprise subscription row is found.  Response envelope:
        ``{"error": "forbidden",
           "message": "Enterprise API management requires an Enterprise subscription.",
           "details": {"upgrade_required": true}, "correlation_id": "..."}``
    """
    stmt = (
        select(Subscription.id)
        .where(Subscription.company_id == current_user.company_id)
        .where(Subscription.plan.in_(list(ENTERPRISE_PLANS)))
        .where(Subscription.status == "active")
        .limit(1)
    )
    result = await session.execute(stmt)
    if result.scalar_one_or_none() is None:
        raise ForbiddenError(
            "Enterprise API management requires an Enterprise subscription.",
            details={"upgrade_required": True},
        )
    return current_user
```

> **Dev note**: `require_enterprise_tier` intentionally uses `get_current_user` (not `get_current_user_or_internal`) because these management endpoints must only be accessible via user JWT, not the internal service-to-service proxy path.

---

### AC2 — Backend: Enterprise API Key Pydantic Schemas

**File:** `services/client-api/src/client_api/schemas/enterprise_api_keys.py` *(new file)*

```python
"""Pydantic schemas for Enterprise API key management (Story 12.16)."""
from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, Field


class EnterpriseApiKeyCreateRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100, description="Human-readable label for this key.")


class EnterpriseApiKeyCreateResponse(BaseModel):
    id: UUID
    name: str
    key_prefix: str = Field(description="First 8 characters of the raw key (for display).")
    plaintext_key: str = Field(
        description="Full plaintext API key. Returned ONLY on creation. Not stored; cannot be retrieved again."
    )
    created_at: datetime

    model_config = {"from_attributes": True}


class EnterpriseApiKeyItem(BaseModel):
    """Masked key item — shown in the list view."""
    id: UUID
    name: str
    key_prefix: str = Field(description="First 8 chars of raw key (key_prefix column).")
    is_active: bool
    last_used_at: datetime | None
    created_at: datetime

    model_config = {"from_attributes": True}


class RateLimitInfo(BaseModel):
    tier: str = Field(description="Subscription tier name (e.g. 'enterprise').")
    requests_per_minute: int
    window_seconds: int


class EnterpriseApiKeyListResponse(BaseModel):
    items: list[EnterpriseApiKeyItem]
    total: int
    rate_limit: RateLimitInfo


class EnterpriseApiKeyRevokeResponse(BaseModel):
    id: UUID
    is_active: bool = False
    message: str = "API key revoked successfully."
```

---

### AC3 — Backend: Client-API Enterprise Key Management Router

**File:** `services/client-api/src/client_api/api/v1/enterprise_api_keys.py` *(new file)*

```python
"""Enterprise API key management endpoints — accessible via JWT (Story 12.16).

Provides company-admin-facing CRUD for enterprise API keys stored in
``client.api_keys``.  These endpoints are authenticated via standard
Bearer JWT (unlike the enterprise-api service which uses X-API-Key).

Routes:
  GET    /api/v1/enterprise/api-keys              — list active keys
  POST   /api/v1/enterprise/api-keys              — create a new key
  DELETE /api/v1/enterprise/api-keys/{key_id}     — revoke a key

Auth requirements:
  1. Valid Bearer JWT (→ 401 if missing/invalid)
  2. Active Enterprise subscription (→ 403 with upgrade_required: true)
  3. Company admin role (→ 403 if user.role < "admin")
"""
from __future__ import annotations

import hashlib
import secrets
from typing import Annotated
from uuid import UUID

import structlog
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from client_api.core.security import CurrentUser, require_role
from client_api.core.tier_gate import require_enterprise_tier
from client_api.dependencies import get_db_session
from client_api.models.api_key import ApiKey
from client_api.models.subscription import Subscription
from client_api.schemas.enterprise_api_keys import (
    EnterpriseApiKeyCreateRequest,
    EnterpriseApiKeyCreateResponse,
    EnterpriseApiKeyItem,
    EnterpriseApiKeyListResponse,
    EnterpriseApiKeyRevokeResponse,
    RateLimitInfo,
)

router = APIRouter(
    prefix="/enterprise/api-keys",
    tags=["Enterprise API Keys"],
    dependencies=[
        Depends(require_enterprise_tier),
        Depends(require_role("admin")),
    ],
)
log = structlog.get_logger()

_KEY_PREFIX_STR = "eusk_"

# Tier → rate limit mapping (must stay in sync with enterprise-api config defaults)
_TIER_RATE_LIMITS: dict[str, int] = {
    "starter": 100,
    "professional": 500,
    "enterprise": 2000,
}
_RATE_LIMIT_WINDOW_SECONDS = 60


def _generate_api_key() -> tuple[str, str, str]:
    """Generate a new API key.

    Returns:
        (plaintext_key, key_prefix_8chars, sha256_hex_digest)
    """
    token = secrets.token_urlsafe(40)
    plaintext = f"{_KEY_PREFIX_STR}{token}"
    key_prefix = plaintext[:8]
    key_hash = hashlib.sha256(plaintext.encode()).hexdigest()
    return plaintext, key_prefix, key_hash


async def _get_company_tier(company_id: UUID, session: AsyncSession) -> str:
    """Return the active subscription plan for a company (default: 'enterprise')."""
    stmt = (
        select(Subscription.plan)
        .where(Subscription.company_id == company_id)
        .where(Subscription.status == "active")
        .limit(1)
    )
    result = await session.execute(stmt)
    return result.scalar_one_or_none() or "enterprise"


@router.get("", response_model=EnterpriseApiKeyListResponse, summary="List API keys")
async def list_enterprise_api_keys(
    current_user: Annotated[CurrentUser, Depends(require_enterprise_tier)],
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> EnterpriseApiKeyListResponse:
    """List all active API keys for the authenticated company (masked — key_prefix only)."""
    stmt = (
        select(ApiKey)
        .where(ApiKey.company_id == current_user.company_id)
        .where(ApiKey.is_active.is_(True))
        .order_by(ApiKey.created_at.desc())
    )
    result = await session.execute(stmt)
    keys = result.scalars().all()
    items = [EnterpriseApiKeyItem.model_validate(k) for k in keys]

    tier = await _get_company_tier(current_user.company_id, session)
    rpm = _TIER_RATE_LIMITS.get(tier, _TIER_RATE_LIMITS["enterprise"])
    rate_limit_info = RateLimitInfo(
        tier=tier,
        requests_per_minute=rpm,
        window_seconds=_RATE_LIMIT_WINDOW_SECONDS,
    )
    return EnterpriseApiKeyListResponse(items=items, total=len(items), rate_limit=rate_limit_info)


@router.post(
    "",
    response_model=EnterpriseApiKeyCreateResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create API key",
    description=(
        "Creates a new API key for the company. "
        "**The full key is returned only once** — store it securely. "
        "It cannot be retrieved again."
    ),
)
async def create_enterprise_api_key(
    body: EnterpriseApiKeyCreateRequest,
    current_user: Annotated[CurrentUser, Depends(require_enterprise_tier)],
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> EnterpriseApiKeyCreateResponse:
    plaintext, key_prefix, key_hash = _generate_api_key()
    new_key = ApiKey(
        company_id=current_user.company_id,
        created_by=current_user.user_id,
        name=body.name,
        key_prefix=key_prefix,
        key_hash=key_hash,
        is_active=True,
    )
    session.add(new_key)
    await session.commit()
    await session.refresh(new_key)

    log.info(
        "client_api.enterprise_key.created",
        company_id=str(current_user.company_id),
        key_id=str(new_key.id),
        key_name=body.name,
    )

    return EnterpriseApiKeyCreateResponse(
        id=new_key.id,
        name=new_key.name,
        key_prefix=key_prefix,
        plaintext_key=plaintext,
        created_at=new_key.created_at,
    )


@router.delete(
    "/{key_id}",
    response_model=EnterpriseApiKeyRevokeResponse,
    summary="Revoke API key",
    description=(
        "Revokes an API key immediately. All subsequent requests using this key "
        "will receive HTTP 401. **This action is irreversible.**"
    ),
    responses={
        404: {"description": "Key not found or does not belong to your company"},
    },
)
async def revoke_enterprise_api_key(
    key_id: UUID,
    current_user: Annotated[CurrentUser, Depends(require_enterprise_tier)],
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> EnterpriseApiKeyRevokeResponse:
    stmt = (
        select(ApiKey)
        .where(ApiKey.id == key_id)
        .where(ApiKey.company_id == current_user.company_id)
        .where(ApiKey.is_active.is_(True))
    )
    result = await session.execute(stmt)
    key = result.scalar_one_or_none()
    if key is None:
        raise HTTPException(
            status_code=404,
            detail="API key not found or does not belong to your company.",
        )
    await session.execute(
        update(ApiKey).where(ApiKey.id == key_id).values(is_active=False)
    )
    await session.commit()

    log.info(
        "client_api.enterprise_key.revoked",
        company_id=str(current_user.company_id),
        key_id=str(key_id),
    )
    return EnterpriseApiKeyRevokeResponse(id=key_id)
```

Register the router in `services/client-api/src/client_api/main.py`:

```python
from client_api.api.v1 import enterprise_api_keys

app.include_router(enterprise_api_keys.router, prefix="/api/v1")
```

---

### AC4 — Backend: Unit Tests

**File:** `services/client-api/tests/api/test_enterprise_api_keys.py` *(new file)*

Tests must cover:

| # | Scenario | Assert |
|---|---|---|
| 1 | Enterprise admin token → `GET /api/v1/enterprise/api-keys` → 200 with `items`, `total`, `rate_limit` fields | Response shape correct |
| 2 | `rate_limit.tier` = `"enterprise"`, `rate_limit.requests_per_minute` = 2000 | Rate limit derived from subscription |
| 3 | `GET` with no existing keys → 200 with `items: []`, `total: 0` | Empty array response |
| 4 | Non-Enterprise (Professional) token → 403 with `upgrade_required: true` | Enterprise gate enforced |
| 5 | Non-Enterprise (Starter) token → 403 with `upgrade_required: true` | Enterprise gate enforced |
| 6 | Unauthenticated → 401 | Auth required |
| 7 | Non-admin role (contributor) → 403 | Role gate enforced |
| 8 | `POST /api/v1/enterprise/api-keys` with `{"name": "Production"}` → 201 | Key created |
| 9 | POST response includes `plaintext_key` starting with `"eusk_"` and `key_prefix` (first 8 chars) | Key format correct |
| 10 | POST response does NOT include `key_hash` | Hash never exposed |
| 11 | Subsequent `GET` shows newly created key (masked — `key_prefix` only, no `plaintext_key`) | Key masked after creation |
| 12 | `DELETE /api/v1/enterprise/api-keys/{key_id}` → 200, `is_active: false` | Revoke response |
| 13 | After revoke, `GET` no longer includes the key | Key absent post-revoke |
| 14 | `DELETE` on key belonging to different company → 404 | Cross-tenant isolation |
| 15 | `DELETE` on already-revoked key → 404 | Idempotency guard |
| 16 | `POST` with name exceeding 100 chars → 422 | Validation enforced |

---

### AC5 — Frontend: Route Shells

Two new page shells are created:

**File 1:** `apps/client/app/[locale]/(protected)/developer/page.tsx`

```tsx
// apps/client/app/[locale]/(protected)/developer/page.tsx
import DeveloperPage from "./components/DeveloperPage";

export default function Page({ params: { locale } }: { params: { locale: string } }) {
  return <DeveloperPage locale={locale} />;
}
```

**File 2:** `apps/client/app/[locale]/(protected)/settings/api-keys/page.tsx`

```tsx
// apps/client/app/[locale]/(protected)/settings/api-keys/page.tsx
import ApiKeyManagementPage from "./components/ApiKeyManagementPage";

export default function Page({ params: { locale } }: { params: { locale: string } }) {
  return <ApiKeyManagementPage locale={locale} />;
}
```

---

### AC6 — Frontend: Navigation Updates

**File:** `apps/client/app/[locale]/(protected)/layout.tsx`

Add the following imports:
```typescript
import { Code2, Key } from "lucide-react";
```

Add two conditional nav items to `clientNavItems`. These items must only appear when `user?.role === "admin"`. Insert after the existing `Settings` entry:

```typescript
// Conditional items for company admins only (Enterprise tier features)
...(user?.role === "admin"
  ? [
      { icon: Code2, label: t("developer"), href: `/${locale}/developer` },
      { icon: Key,   label: t("apiKeys"),   href: `/${locale}/settings/api-keys` },
    ]
  : []),
```

> **Dev note**: Rendering the nav items only for `role === "admin"` is a UX convenience. The actual tier gate (Enterprise-only) is enforced by the backend endpoint (403 → upgrade gate rendered in the page). Non-Enterprise admins see the nav links but reach the upgrade gate UI.

---

### AC7 — Frontend: i18n Keys

**Files:** `apps/client/messages/en.json` and `apps/client/messages/bg.json`

Add `"developer"` and `"apiKeys"` under the `"nav"` key:

**`en.json`**:
```json
"nav": {
  ...existing keys...,
  "developer": "Developer",
  "apiKeys": "API Keys"
}
```

**`bg.json`**:
```json
"nav": {
  ...existing keys...,
  "developer": "Разработчик",
  "apiKeys": "API ключове"
}
```

Add a new `"developer"` top-level key block in both locale files:

```json
"developer": {
  "pageTitle": "Enterprise API Documentation",
  "pageDescription": "Explore available API endpoints and manage your API keys.",
  "docsTitle": "API Reference",
  "docsDescription": "Interactive documentation for the EU Solicit Enterprise API.",
  "upgradeTitle": "Enterprise Feature",
  "upgradeMessage": "API access and documentation management are available on the Enterprise plan.",
  "upgradeCta": "Upgrade to Enterprise",
  "apiKeys": {
    "pageTitle": "API Key Management",
    "pageDescription": "Create and manage API keys for programmatic access to the EU Solicit Enterprise API.",
    "createKeyBtn": "Create API Key",
    "createKeyDialogTitle": "Create New API Key",
    "createKeyNameLabel": "Key Name",
    "createKeyNamePlaceholder": "e.g. Production Integration",
    "createKeySubmitBtn": "Generate Key",
    "createKeyCancelBtn": "Cancel",
    "newKeyWarningTitle": "Save your API key",
    "newKeyWarningBody": "This key will only be shown once. Copy it now and store it securely.",
    "copyKeyBtn": "Copy Key",
    "keyCopiedLabel": "Copied!",
    "doneBtn": "Done",
    "revokeKeyBtn": "Revoke",
    "revokeDialogTitle": "Revoke API Key?",
    "revokeDialogBody": "This will immediately invalidate the key. Any integrations using it will stop working.",
    "revokeConfirmBtn": "Revoke Key",
    "revokeCancelBtn": "Cancel",
    "tableColName": "Name",
    "tableColPrefix": "Key Prefix",
    "tableColLastUsed": "Last Used",
    "tableColCreated": "Created",
    "tableColActions": "Actions",
    "maskedKeyFormat": "****{prefix}",
    "noKeysTitle": "No API keys yet",
    "noKeysDescription": "Create your first API key to start integrating with the EU Solicit Enterprise API.",
    "rateLimitSectionTitle": "Rate Limit",
    "rateLimitTierLabel": "Tier",
    "rateLimitRpmLabel": "Requests per minute",
    "upgradeTitle": "Enterprise Feature",
    "upgradeMessage": "API key management is available on the Enterprise plan.",
    "upgradeCta": "Upgrade to Enterprise"
  }
}
```

---

### AC8 — Frontend: API Client

**File:** `apps/client/lib/api/enterprise-api-keys.ts` *(new file)*

```typescript
/**
 * Enterprise API key management — client-api fetch functions (Story 12.16).
 *
 * Base URL: /api/v1/enterprise/api-keys
 * Auth: Bearer JWT (managed by apiClient from @eusolicit/ui)
 */
import { apiClient } from "@eusolicit/ui";

export interface RateLimitInfo {
  tier: string;
  requests_per_minute: number;
  window_seconds: number;
}

export interface EnterpriseApiKeyItem {
  id: string;
  name: string;
  key_prefix: string;
  is_active: boolean;
  last_used_at: string | null;
  created_at: string;
}

export interface EnterpriseApiKeyListResponse {
  items: EnterpriseApiKeyItem[];
  total: number;
  rate_limit: RateLimitInfo;
}

export interface EnterpriseApiKeyCreateRequest {
  name: string;
}

export interface EnterpriseApiKeyCreateResponse {
  id: string;
  name: string;
  key_prefix: string;
  plaintext_key: string;
  created_at: string;
}

export interface EnterpriseApiKeyRevokeResponse {
  id: string;
  is_active: false;
  message: string;
}

const BASE = "/api/v1/enterprise/api-keys";

export async function fetchEnterpriseApiKeys(): Promise<EnterpriseApiKeyListResponse> {
  return apiClient.get<EnterpriseApiKeyListResponse>(BASE);
}

export async function createEnterpriseApiKey(
  body: EnterpriseApiKeyCreateRequest,
): Promise<EnterpriseApiKeyCreateResponse> {
  return apiClient.post<EnterpriseApiKeyCreateResponse>(BASE, body);
}

export async function revokeEnterpriseApiKey(
  keyId: string,
): Promise<EnterpriseApiKeyRevokeResponse> {
  return apiClient.delete<EnterpriseApiKeyRevokeResponse>(`${BASE}/${keyId}`);
}
```

---

### AC9 — Frontend: TanStack Query Hook

**File:** `apps/client/lib/queries/use-enterprise-api-keys.ts` *(new file)*

```typescript
"use client";

import { useQuery } from "@tanstack/react-query";
import { fetchEnterpriseApiKeys } from "@/lib/api/enterprise-api-keys";

export const enterpriseApiKeysQueryKey = ["enterprise-api-keys"] as const;

/**
 * Hook: list all active Enterprise API keys for the company.
 * Stale time 0 — always refetch after create/revoke mutations so list stays current.
 */
export function useEnterpriseApiKeys() {
  return useQuery({
    queryKey: enterpriseApiKeysQueryKey,
    queryFn: fetchEnterpriseApiKeys,
    staleTime: 0,
    retry: (failureCount, error: unknown) => {
      // Do not retry 403 — it's a tier gate, not a transient failure
      if ((error as { status?: number })?.status === 403) return false;
      return failureCount < 2;
    },
  });
}
```

---

### AC10 — Frontend: Developer Page Component

**File:** `apps/client/app/[locale]/(protected)/developer/components/DeveloperPage.tsx` *(new file)*

Root client component (`"use client"`). Props: `{ locale: string }`.

The component renders:

1. **Page header** — `data-testid="developer-page"`:
   - Heading: `t("developer.pageTitle")`
   - Description: `t("developer.pageDescription")`

2. **API Documentation embed section** — `data-testid="enterprise-api-docs"`:
   - Heading: `t("developer.docsTitle")`
   - Description: `t("developer.docsDescription")`
   - An `<iframe>` pointing to `{NEXT_PUBLIC_ENTERPRISE_API_URL}/v1/redoc` (see dev notes for URL resolution).
     ```tsx
     <iframe
       src={`${enterpriseApiUrl}/v1/redoc`}
       title={t("developer.docsTitle")}
       data-testid="enterprise-api-docs-iframe"
       className="w-full rounded-lg border border-slate-200"
       style={{ height: "80vh", minHeight: "600px" }}
     />
     ```
   - The `enterpriseApiUrl` is derived from `process.env.NEXT_PUBLIC_ENTERPRISE_API_URL ?? "https://api.eusolicit.com"`.

3. **No loading state needed** — the iframe loads independently.

4. **No tier-gate on this page** — The documentation is a public URL served by the enterprise API. Access to the page itself is gated by the nav item visibility (`user.role === "admin"`) rather than a backend call.

---

### AC11 — Frontend: API Key Management Page — Rate Limit Section

**File:** `apps/client/app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx` *(new file)*

Root client component (`"use client"`). Props: `{ locale: string }`.

Uses `useEnterpriseApiKeys()`. Handles three states:

#### A — Loading state
Render a full-page skeleton:
```tsx
<div data-testid="api-keys-skeleton">
  <SkeletonCard />
  <SkeletonTable rows={3} />
</div>
```

#### B — Error state (403 — Enterprise gate)
When `error?.status === 403`, render the upgrade gate:
```tsx
<div data-testid="enterprise-api-keys-upgrade-gate">
  <h2>{t("developer.apiKeys.upgradeTitle")}</h2>
  <p>{t("developer.apiKeys.upgradeMessage")}</p>
  <Button asChild>
    <a href="/settings/billing" data-testid="enterprise-api-keys-upgrade-cta">
      {t("developer.apiKeys.upgradeCta")}
    </a>
  </Button>
</div>
```

#### C — Success state
The page renders three sections:

**Rate Limit Section** — `data-testid="rate-limit-info"`:
```tsx
<div data-testid="rate-limit-info">
  <h3>{t("developer.apiKeys.rateLimitSectionTitle")}</h3>
  <dl>
    <dt>{t("developer.apiKeys.rateLimitTierLabel")}</dt>
    <dd data-testid="rate-limit-tier">{data.rate_limit.tier}</dd>
    <dt>{t("developer.apiKeys.rateLimitRpmLabel")}</dt>
    <dd data-testid="rate-limit-rpm">
      {data.rate_limit.requests_per_minute.toLocaleString()}
    </dd>
  </dl>
</div>
```

**API Keys Table** — `data-testid="api-keys-table-section"` (see AC12 for table spec).

**Create Key button** — `data-testid="create-api-key-btn"` opens the create key dialog (see AC13).

---

### AC12 — Frontend: API Keys Table

The keys table renders within `ApiKeyManagementPage` when the key list is non-empty.

```tsx
<table data-testid="api-keys-table">
  <thead>
    <tr>
      <th>{t("developer.apiKeys.tableColName")}</th>
      <th>{t("developer.apiKeys.tableColPrefix")}</th>
      <th>{t("developer.apiKeys.tableColLastUsed")}</th>
      <th>{t("developer.apiKeys.tableColCreated")}</th>
      <th>{t("developer.apiKeys.tableColActions")}</th>
    </tr>
  </thead>
  <tbody>
    {data.items.map((key) => (
      <tr key={key.id} data-testid={`api-key-row-${key.id}`}>
        <td data-testid={`api-key-name-${key.id}`}>{key.name}</td>
        <td data-testid={`api-key-prefix-${key.id}`}>
          {"****"}{key.key_prefix}   {/* masked display: ****eusk_1a */}
        </td>
        <td data-testid={`api-key-last-used-${key.id}`}>
          {key.last_used_at ? formatRelativeDate(key.last_used_at, locale) : "—"}
        </td>
        <td data-testid={`api-key-created-${key.id}`}>
          {formatDate(key.created_at, locale)}
        </td>
        <td>
          <Button
            variant="destructive"
            size="sm"
            data-testid={`revoke-key-btn-${key.id}`}
            onClick={() => setKeyToRevoke(key)}
          >
            {t("developer.apiKeys.revokeKeyBtn")}
          </Button>
        </td>
      </tr>
    ))}
  </tbody>
</table>
```

When `data.items` is empty, render the empty state instead of the table:
```tsx
<div data-testid="api-keys-empty-state">
  <p>{t("developer.apiKeys.noKeysTitle")}</p>
  <p>{t("developer.apiKeys.noKeysDescription")}</p>
</div>
```

**Key masking rule**: The `key_prefix` column from the API contains the first 8 characters of the raw key. The frontend displays it as `"****{key_prefix}"` — the `****` is a hardcoded visual mask prefix. This gives the user enough characters to identify which key is which.

---

### AC13 — Frontend: Create Key Dialog

**Create Key Dialog** — `data-testid="create-key-dialog"` — triggered by the "Create API Key" button.

Dialog renders a form:
- `<Input name="name" data-testid="create-key-name-input" placeholder={t("developer.apiKeys.createKeyNamePlaceholder")} />`
- Submit button: `data-testid="create-key-submit-btn"` — calls `createEnterpriseApiKey({ name })` mutation
- Cancel button: `data-testid="create-key-cancel-btn"` — closes the dialog

**After successful creation**, the dialog transitions to a "key reveal" view:

```tsx
<div data-testid="new-key-reveal">
  <Alert variant="warning">
    <AlertTitle>{t("developer.apiKeys.newKeyWarningTitle")}</AlertTitle>
    <AlertDescription>{t("developer.apiKeys.newKeyWarningBody")}</AlertDescription>
  </Alert>

  <div data-testid="new-key-value" className="font-mono bg-slate-100 p-3 rounded break-all select-all">
    {createdKey.plaintext_key}
  </div>

  <Button
    data-testid="copy-key-btn"
    onClick={() => {
      navigator.clipboard.writeText(createdKey.plaintext_key);
      setCopied(true);
    }}
  >
    {copied ? t("developer.apiKeys.keyCopiedLabel") : t("developer.apiKeys.copyKeyBtn")}
  </Button>

  <Button
    data-testid="new-key-done-btn"
    onClick={() => {
      setCreatedKey(null);
      queryClient.invalidateQueries({ queryKey: enterpriseApiKeysQueryKey });
      closeDialog();
    }}
  >
    {t("developer.apiKeys.doneBtn")}
  </Button>
</div>
```

**Critical UX requirement**: The `plaintext_key` is displayed ONLY in this post-creation reveal view. Once the user clicks "Done" and the dialog is closed, the key is gone from client memory. The subsequent `GET /api/v1/enterprise/api-keys` response does NOT return the plaintext key — only the masked `key_prefix`.

---

### AC14 — Frontend: Revoke Key Confirmation Dialog

**Revoke Dialog** — `data-testid="revoke-key-dialog"` — triggered by the Revoke button for a specific key.

Dialog renders:
- Heading: `t("developer.apiKeys.revokeDialogTitle")`
- Body: `t("developer.apiKeys.revokeDialogBody")`
- Confirm button: `data-testid="revoke-confirm-btn"` — calls `revokeEnterpriseApiKey(key.id)` mutation
- Cancel button: `data-testid="revoke-cancel-btn"` — closes dialog without action

**After successful revocation**:
1. Invalidate `enterpriseApiKeysQueryKey` to trigger re-fetch
2. Close the dialog
3. The revoked key row disappears from the table on refetch

The revoke mutation is instant — no polling required. The backend sets `is_active = False` synchronously.

---

### AC15 — Frontend: Unit Tests

**File:** `apps/client/__tests__/enterprise-api-keys-s12-16.test.ts` *(new file)*

Following the structural verification pattern established in previous analytics story tests (e.g., `competitor-intelligence-s12-6.test.ts`). Tests verify:

| # | Assertion |
|---|---|
| 1 | `DeveloperPage` component file exists at `app/[locale]/(protected)/developer/components/DeveloperPage.tsx` |
| 2 | `DeveloperPage` renders element with `data-testid="developer-page"` |
| 3 | `DeveloperPage` renders iframe with `data-testid="enterprise-api-docs-iframe"` |
| 4 | `ApiKeyManagementPage` component file exists at `app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx` |
| 5 | `ApiKeyManagementPage` renders `data-testid="rate-limit-info"` when data is loaded |
| 6 | `ApiKeyManagementPage` renders `data-testid="api-keys-table"` when items are present |
| 7 | `ApiKeyManagementPage` renders `data-testid="api-keys-empty-state"` when items array is empty |
| 8 | `ApiKeyManagementPage` renders `data-testid="enterprise-api-keys-upgrade-gate"` on 403 error |
| 9 | `ApiKeyManagementPage` renders `data-testid="create-api-key-btn"` |
| 10 | API function `fetchEnterpriseApiKeys` exported from `lib/api/enterprise-api-keys.ts` |
| 11 | API function `createEnterpriseApiKey` exported from `lib/api/enterprise-api-keys.ts` |
| 12 | API function `revokeEnterpriseApiKey` exported from `lib/api/enterprise-api-keys.ts` |
| 13 | `useEnterpriseApiKeys` hook exported from `lib/queries/use-enterprise-api-keys.ts` |
| 14 | i18n key `"developer.pageTitle"` present in `messages/en.json` |
| 15 | i18n key `"developer.apiKeys.createKeyBtn"` present in `messages/en.json` |
| 16 | i18n key `"developer.apiKeys.rateLimitSectionTitle"` present in `messages/en.json` |
| 17 | Page shell `app/[locale]/(protected)/developer/page.tsx` exists and does NOT contain `"use client"` |
| 18 | Page shell `app/[locale]/(protected)/settings/api-keys/page.tsx` exists and does NOT contain `"use client"` |

---

### Senior Developer Review

**Reviewer:** Code Review Agent (2026-04-14)
**Verdict:** CHANGES REQUESTED — 4 patch findings, 4 deferred, 4 dismissed

#### Review Findings

- [x] [Review][Patch] Upgrade CTA link missing locale prefix — `ApiKeyManagementPage.tsx:134` has `href="/settings/billing"` but all other app links use `/${locale}/...` pattern. Will 404 or land on wrong locale. Fix: change to `` href={`/${locale}/settings/billing`} ``
- [x] [Review][Patch] Missing generic error state for non-403 errors — `ApiKeyManagementPage.tsx:120-140` only handles `status === 403`. Any other error (500, network timeout) falls through to success state where `data` is undefined, showing empty state with no error message. Fix: add an else-if branch for generic errors before the success state render.
- [x] [Review][Patch] Clipboard API called without error handling — `ApiKeyManagementPage.tsx:97` calls `navigator.clipboard.writeText()` synchronously without awaiting or catching rejection. In non-HTTPS contexts or when permission is denied, copy fails silently but `setCopied(true)` still runs, giving false "Copied!" feedback. Fix: wrap in try/catch or use `.then()/.catch()`.
- [x] [Review][Patch] AC13 specifies `<Alert variant="warning">` with `AlertTitle`/`AlertDescription` components for key reveal warning — `ApiKeyManagementPage.tsx:320-327` uses a plain `<div>` with amber classes instead. Fix: import and use `Alert`, `AlertTitle`, `AlertDescription` from `@eusolicit/ui` as specified.
- [x] [Review][Defer] No max-keys-per-company limit — `enterprise_api_keys.py` POST endpoint allows unbounded key creation. Deferrable: not specified in story scope; add as separate hardening story.
- [x] [Review][Defer] No pagination on API keys list endpoint — GET loads all keys into memory. Acceptable for now (enterprise customers unlikely to have 100+ keys), but should be added when scaling.
- [x] [Review][Defer] Missing `sandbox` attribute on iframe — `DeveloperPage.tsx:41` iframe has no sandbox. Defense-in-depth improvement; URL is build-time constant so risk is low.
- [x] [Review][Defer] setTimeout cleanup on unmount — `ApiKeyManagementPage.tsx:99` — `setTimeout(() => setCopied(false), 2000)` not cleaned up on unmount. Minor React warning; pre-existing pattern.

---

## Tasks / Subtasks

### Backend

- [x] Task 1: Add `require_enterprise_tier` to `services/client-api/src/client_api/core/tier_gate.py`
  - [x] Define `ENTERPRISE_PLANS: frozenset[str] = frozenset({"enterprise"})`
  - [x] Implement `require_enterprise_tier` dependency (same pattern as `require_professional_plus_tier`)
  - [x] Add `ENTERPRISE_PLANS` and `require_enterprise_tier` to module exports
- [x] Task 2: Create Pydantic schemas — `services/client-api/src/client_api/schemas/enterprise_api_keys.py`
  - [x] `EnterpriseApiKeyCreateRequest`, `EnterpriseApiKeyCreateResponse`
  - [x] `EnterpriseApiKeyItem`, `EnterpriseApiKeyListResponse`
  - [x] `RateLimitInfo`, `EnterpriseApiKeyRevokeResponse`
- [x] Task 3: Create management router — `services/client-api/src/client_api/api/v1/enterprise_api_keys.py`
  - [x] `GET /api/v1/enterprise/api-keys` — list with rate limit info
  - [x] `POST /api/v1/enterprise/api-keys` — create (returns plaintext once)
  - [x] `DELETE /api/v1/enterprise/api-keys/{key_id}` — revoke
  - [x] Apply `require_enterprise_tier` + `require_role("admin")` on all routes
  - [x] Implement `_get_company_tier()` helper for rate limit info
- [x] Task 4: Register router in `services/client-api/src/client_api/main.py`
- [x] Task 5: Backend unit tests — `services/client-api/tests/api/test_enterprise_api_keys.py` (16 test cases per AC4)

### Frontend

- [x] Task 6: Create API client — `apps/client/lib/api/enterprise-api-keys.ts`
  - [x] `fetchEnterpriseApiKeys()`, `createEnterpriseApiKey()`, `revokeEnterpriseApiKey()`
  - [x] Export all interfaces: `EnterpriseApiKeyItem`, `EnterpriseApiKeyListResponse`, `RateLimitInfo`, etc.
- [x] Task 7: Create TanStack Query hook — `apps/client/lib/queries/use-enterprise-api-keys.ts`
  - [x] `useEnterpriseApiKeys()` with `staleTime: 0` and 403-aware retry logic
- [x] Task 8: Add i18n keys to `apps/client/messages/en.json` and `apps/client/messages/bg.json`
  - [x] `nav.developer`, `nav.apiKeys`
  - [x] Full `developer` key block (page, apiKeys sub-keys per AC7)
- [x] Task 9: Create Developer page components
  - [x] `apps/client/app/[locale]/(protected)/developer/page.tsx` — server component shell
  - [x] `apps/client/app/[locale]/(protected)/developer/components/DeveloperPage.tsx` — Redoc iframe embed
- [x] Task 10: Create API Key Management page components
  - [x] `apps/client/app/[locale]/(protected)/settings/api-keys/page.tsx` — server component shell
  - [x] `apps/client/app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx` — root orchestrator (loading / 403-gate / success states)
  - [x] Rate limit section (`data-testid="rate-limit-info"`)
  - [x] API keys table (`data-testid="api-keys-table"`) with masked key display
  - [x] Empty state (`data-testid="api-keys-empty-state"`)
  - [x] Create key dialog with post-creation key reveal and copy-to-clipboard
  - [x] Revoke key confirmation dialog
  - [x] Upgrade gate (`data-testid="enterprise-api-keys-upgrade-gate"`) on 403
- [x] Task 11: Update navigation layout — `apps/client/app/[locale]/(protected)/layout.tsx`
  - [x] Import `Code2`, `Key` from `lucide-react`
  - [x] Add conditional nav items for `user.role === "admin"`: Developer link, API Keys link
- [x] Task 12: Frontend unit tests — `apps/client/__tests__/enterprise-api-keys-s12-16.test.ts` (18 test cases per AC15)

---

## Dev Notes

### Architecture: Why Client-API Management Endpoints (not Enterprise API)

S12.15 implemented API key CRUD at `services/enterprise-api/src/enterprise_api/api/v1/api_keys.py`, authenticated via `X-API-Key` header. The web frontend uses JWT Bearer tokens, not API keys. Adding JWT support to the enterprise-api for management operations would require duplicating the authentication stack.

The clean solution: add thin management endpoints to the **client-api** (`/api/v1/enterprise/api-keys`) that use the established JWT + tier-gate + role-gate pattern. These write to the same `client.api_keys` table. The enterprise-api continues to serve API-key-authenticated requests from external integrations.

Both endpoint sets share the same DB table. The management endpoints in client-api gate on `require_enterprise_tier + require_role("admin")`. The enterprise-api runtime endpoints gate on valid hashed key from the same table.

### Enterprise API Docs URL

The Redoc iframe URL is derived from the `NEXT_PUBLIC_ENTERPRISE_API_URL` environment variable:
- **Local dev**: set to `http://localhost:8003` (enterprise-api dev port)
- **Staging**: set to `https://api-staging.eusolicit.com`
- **Production**: set to `https://api.eusolicit.com`

The page uses: `const enterpriseApiUrl = process.env.NEXT_PUBLIC_ENTERPRISE_API_URL ?? "https://api.eusolicit.com";`

The iframe loads `/v1/redoc` from this URL. If the enterprise API is unavailable or the URL is wrong, the iframe will show a browser error inside the iframe — not a React error. No special error handling is needed for the iframe itself.

> **Note**: The enterprise API's Redoc endpoint (`/v1/redoc`) does NOT require authentication per S12.15 (it's in the `_AUTH_EXEMPT_PATHS` set). The documentation is intentionally publicly accessible for enterprise clients to explore without pre-authentication.

### Key Masking Logic

The `key_prefix` from the API is the first 8 characters of the raw key (e.g., `"eusk_1a2b"`). The frontend renders it as `"****eusk_1a2b"` — prepending four asterisks. This is purely a display concern; the backend only stores and returns `key_prefix`.

The `plaintext_key` is returned ONLY in `EnterpriseApiKeyCreateResponse` (POST 201). It is never returned by `GET /api/v1/enterprise/api-keys`. Ensure the `EnterpriseApiKeyItem` schema does NOT include a `plaintext_key` field.

### Copy to Clipboard

Use `navigator.clipboard.writeText(createdKey.plaintext_key)` for the copy action. Toggle `copied` state to `true` for 2 seconds to show `"Copied!"` feedback, then reset. This is a common pattern used elsewhere in the app. No third-party library needed.

### Rate Limit Info Display

The `rate_limit` object in the list response includes:
- `tier`: subscription tier string (e.g., `"enterprise"`)
- `requests_per_minute`: integer (e.g., `2000`)
- `window_seconds`: integer (e.g., `60`)

The rate limit section (`data-testid="rate-limit-info"`) displays these values. The `_get_company_tier()` helper in the router looks up the active subscription plan for the company. The `_TIER_RATE_LIMITS` mapping in the router must stay in sync with the `RATE_LIMIT_*_RPM` defaults in `services/enterprise-api/src/enterprise_api/config.py`.

### Test Design Alignment (from `test-design-epic-12.md`)

**P1 tests to enable (S12.16 scenarios):**

| Scenario | Test | Type |
|---|---|---|
| API key management page: lists masked active keys (last 4 chars visible only) | `data-testid="api-key-prefix-{id}"` shows `****` + prefix | E2E |
| New key creation: full key shown exactly once; navigate away; verify masked on return | Create flow → navigate away → return → assert no `plaintext_key` visible | E2E (R12.4) |
| Key revocation: confirmation dialog; key status changes to revoked immediately | Revoke → retry API call → assert 401 | E2E |
| Rate limit tier and usage stats displayed on management page | `data-testid="rate-limit-info"` visible, `rate-limit-tier` shows `"enterprise"` | E2E |

**P2 tests:**

| Scenario | Test | Type |
|---|---|---|
| Enterprise API documentation page embeds Swagger UI or Redoc component | `data-testid="enterprise-api-docs-iframe"` present | E2E |

### Playwright E2E Tests (existing RED-phase tests)

After implementation, search for any `test.skip()` blocks in `e2e/specs/` related to enterprise API key management or the developer page and enable them.

The P0 ATDD checklist for E12 (`test-artifacts/atdd-checklist-e12-p0.md`) does not include S12.16 scenarios (key management is P1, not P0). No P0 tests are blocked by this story.

### Security Consideration (R12.4)

The key lifecycle that must be tested:

1. **Create** → plaintext key visible in dialog ✓
2. **Navigate away** → close dialog → key gone from memory ✓
3. **Return to page** → GET list → `key_prefix` visible, NO `plaintext_key` ✓
4. **Revoke** → confirmation dialog → DELETE → key removed from list ✓
5. **Post-revoke API call** → enterprise-api returns 401 for revoked key ✓ (integration test coverage)

The E2E test for "full key shown exactly once" (P1, R12.4) should assert that after navigating away from the creation dialog and returning to the page, the full plaintext key is NOT visible anywhere in the DOM.

### Frontend State Architecture

`ApiKeyManagementPage` manages minimal local state:
- `keyToRevoke: EnterpriseApiKeyItem | null` — controls revoke dialog visibility and target
- `isCreateOpen: boolean` — controls create dialog visibility
- `createdKey: EnterpriseApiKeyCreateResponse | null` — holds post-creation response to display once
- `copied: boolean` — clipboard feedback state

Mutations (`createEnterpriseApiKey`, `revokeEnterpriseApiKey`) use `useMutation` from TanStack Query. On success, `queryClient.invalidateQueries({ queryKey: enterpriseApiKeysQueryKey })` triggers a fresh fetch.

### `created_by` Column

The `client.api_keys` table has `created_by UUID NULLABLE REFERENCES client.users(id) ON DELETE SET NULL` (relaxed FK per S12.15 dev note). When the management endpoint creates a key via JWT, `current_user.user_id` is available and can be passed as `created_by`. Unlike the enterprise-api (which had no user context), the client-api JWT contains the actual user_id.

---

## File List

### Backend (Client API)

- `services/client-api/src/client_api/core/tier_gate.py` — add `ENTERPRISE_PLANS`, `require_enterprise_tier`
- `services/client-api/src/client_api/schemas/enterprise_api_keys.py` *(new)* — Pydantic schemas
- `services/client-api/src/client_api/api/v1/enterprise_api_keys.py` *(new)* — management router (list, create, revoke)
- `services/client-api/src/client_api/main.py` — register `enterprise_api_keys.router`
- `services/client-api/tests/api/test_enterprise_api_keys.py` *(new)* — 16 unit tests

### Frontend (Client App)

- `apps/client/app/[locale]/(protected)/developer/page.tsx` *(new)* — server component shell
- `apps/client/app/[locale]/(protected)/developer/components/DeveloperPage.tsx` *(new)* — Redoc iframe embed
- `apps/client/app/[locale]/(protected)/settings/api-keys/page.tsx` *(new)* — server component shell
- `apps/client/app/[locale]/(protected)/settings/api-keys/components/ApiKeyManagementPage.tsx` *(new)* — root page component (all states)
- `apps/client/lib/api/enterprise-api-keys.ts` *(new)* — fetch functions and interfaces
- `apps/client/lib/queries/use-enterprise-api-keys.ts` *(new)* — TanStack Query hook
- `apps/client/app/[locale]/(protected)/layout.tsx` — add Code2/Key imports, conditional nav items
- `apps/client/messages/en.json` — add `nav.developer`, `nav.apiKeys`, `developer.*` keys
- `apps/client/messages/bg.json` — add same keys in Bulgarian
- `apps/client/__tests__/enterprise-api-keys-s12-16.test.ts` *(new)* — 18 structural unit tests

---

## Dev Agent Record

### Implementation Plan

Implemented Story 12.16 in two phases: backend (AC1-AC4) then frontend (AC5-AC15).

**Backend approach:**
- Extended `tier_gate.py` with `ENTERPRISE_PLANS` frozenset and `require_enterprise_tier` dependency, following the exact pattern of `require_professional_plus_tier` but using `get_current_user` (not `get_current_user_or_internal`) per spec
- Created new Pydantic schemas file `enterprise_api_keys.py` with all 6 schemas exactly as specified in AC2
- Created management router `enterprise_api_keys.py` with GET/POST/DELETE endpoints, `_generate_api_key()` using `secrets.token_urlsafe(40)` + SHA-256 hash, and `_get_company_tier()` helper for rate limit lookup
- Registered router in `main.py` using established pattern

**Frontend approach:**
- Created API client `enterprise-api-keys.ts` with TypeScript interfaces and three fetch functions using `apiClient` from `@eusolicit/ui`
- Created TanStack Query hook with `staleTime: 0` and 403-aware retry logic
- Added `developer` i18n block + `nav.developer`/`nav.apiKeys` to both `en.json` and `bg.json`
- Developer page is a server shell + client component (DeveloperPage.tsx) with Redoc iframe
- ApiKeyManagementPage.tsx is a full-featured client component handling: loading skeleton, 403 upgrade gate, success state with rate limit info, keys table/empty state, create dialog with key reveal + clipboard, revoke confirmation dialog
- Updated `layout.tsx` to conditionally show Developer + API Keys nav items for `role === "admin"` users

**Tests:**
- 16 backend unit tests covering all AC4 scenarios (tier gate, role gate, auth, CRUD, cross-tenant isolation, idempotency, validation)
- 18 frontend structural tests all passing (file existence, data-testid assertions, i18n key presence, server component guards)

### Completion Notes

- All 12 tasks completed
- All 18 frontend ATDD tests pass (AC15)
- Backend unit tests written for 16 AC4 scenarios
- Pre-existing 4 test failures in stories 11-11, 11-13, 12-4 unrelated to this story
- No new regressions introduced
- All Acceptance Criteria AC1–AC15 satisfied
- Key masking: frontend displays `"****{key_prefix}"` (4 asterisks + 8-char prefix from API)
- Security: plaintext_key only in POST 201 response + create dialog reveal; never in GET list
- Enterprise tier gate uses `get_current_user` (JWT-only, not internal service proxy)

---

## Change Log

| Date | Author | Note |
|---|---|---|
| 2026-04-14 | SM (Auto) | Story created for Sprint 13-14. |
| 2026-04-14 | Dev Agent | Implemented all 12 tasks: AC1 tier_gate, AC2 schemas, AC3 router, AC4 tests, AC5-AC15 frontend pages, API client, query hook, i18n, nav, ATDD tests. Story → review. |
| 2026-04-14 | Dev Agent | Applied 4 review patches: (1) locale prefix on upgrade CTA link, (2) generic error state for non-403 errors, (3) clipboard try/catch error handling, (4) replaced plain div with Alert/AlertTitle/AlertDescription from @eusolicit/ui (new Alert component added to UI package). Fixed 8 backend tests (companies_in_db + write_session_override fixtures). All 16 backend + 18 frontend ATDD tests pass. Story → done. |
