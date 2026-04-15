# Security Audit Checklist — EU Solicit Pre-Launch (Story 12.17)

**Date:** _YYYY-MM-DD_
**Auditor:** _Name/Role_
**Status:** ☐ In Progress  ☐ Complete  ☐ Signed Off

---

## OWASP Top 10 (2021)

### A01 — Broken Access Control

| Check | Evidence | Status |
|:---|:---|:---:|
| All admin endpoints return 403 for non-VPN IPs (IPAllowlistMiddleware verified in staging) | `tests/e2e/admin/` P0 test run | ☐ |
| Standard user JWT cannot access other users' analytics data (`company_id` scoping on all MV queries) | Cross-tenant P0 API tests | ☐ |
| Tier gates return 403 + upgrade message on Professional+ gated endpoints | `tests/e2e/analytics/` P0 tier tests | ☐ |
| Enterprise API key revocation takes effect immediately (revoked key → 401) | `tests/e2e/enterprise/` P0 lifecycle test | ☐ |
| Company admin route guard blocks non-admin role from admin frontend pages | `tests/e2e/admin/` E2E P0 route guard test | ☐ |

### A02 — Cryptographic Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| All client-to-service traffic uses TLS 1.2+ (nginx `ssl_protocols TLSv1.2 TLSv1.3`) | `infra/nginx/eusolicit.com` reviewed | ☐ |
| JWT uses RS256 (not HS256); private key stored in Kubernetes Secret, not ConfigMap | `services/client-api/src/client_api/core/security.py` reviewed | ☐ |
| Enterprise API keys stored as SHA-256 hashes in DB; plaintext never persisted | `services/client-api/alembic/versions/015_enterprise_api_keys.py` + key creation code reviewed | ☐ |
| DB passwords use strong random strings (≥ 32 chars); no default passwords in production | Secret manager entry verified (see §Secret Rotation) | ☐ |
| S3 report bucket not publicly accessible; only signed URLs used | AWS S3 bucket policy verified | ☐ |

### A03 — Injection

| Check | Evidence | Status |
|:---|:---|:---:|
| All analytics queries use SQLAlchemy Core parameterised statements (no raw f-string SQL) | Code review: `services/client-api/src/client_api/services/analytics_*_service.py` | ☐ |
| Admin API queries use parameterised ORM (no raw SQL concatenation) | Code review: `services/admin-api/src/admin_api/services/` | ☐ |
| All Pydantic input models validate types and lengths (no unvalidated user strings in SQL filters) | Spot-check of request schemas in `schemas/` directories | ☐ |

### A04 — Insecure Design

| Check | Evidence | Status |
|:---|:---|:---:|
| Rate limiting applied at ingress layer (nginx `limit-rps: "20"`) and application layer (Redis token bucket for enterprise-api) | `infra/helm/values/client-api.yaml` + `enterprise-api/middleware/rate_limit.py` reviewed | ☐ |
| Login rate limiting (5 failures / 15 min) prevents brute force on auth endpoint | `services/client-api/src/client_api/core/rate_limit.py` reviewed | ☐ |
| Multi-tenant data isolation enforced at SQL layer (`WHERE company_id = :cid`) in all analytics services | All `analytics_*_service.py` files reviewed | ☐ |

### A05 — Security Misconfiguration

| Check | Evidence | Status |
|:---|:---|:---:|
| CORS origins restricted to known frontend origins (`cors_origins_list` from env var, not `*`) | `services/client-api/src/client_api/main.py` CORSMiddleware config | ☐ |
| Kubernetes network policies enabled for all 6 services (admin-api, ai-gateway, client-api, enterprise-api, notification, data-pipeline) | Helm values reviewed; `kubectl get networkpolicy -A` output attached | ☐ |
| No debug endpoints (`/debug`, `/admin/shell`) exposed in production | `main.py` files reviewed; ingress rules confirmed | ☐ |
| Swagger UI (`/v1/docs`) and Redoc (`/v1/redoc`) are enterprise-api only — not exposed on client-api or admin-api | `client_api/main.py` and `admin_api/main.py` — no `docs_url` set | ☐ |

### A06 — Vulnerable and Outdated Components

| Check | Evidence | Status |
|:---|:---|:---:|
| `pip-audit` run against all service `pyproject.toml`; no HIGH/CRITICAL CVEs | `pip-audit` output attached | ☐ |
| `npm audit` run against `frontend/package.json`; no HIGH/CRITICAL CVEs | `npm audit --json` output attached | ☐ |
| Base Docker images pinned to specific digest tags (not `:latest`) in production Dockerfiles | `Dockerfile` files reviewed | ☐ |

### A07 — Identification and Authentication Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| JWT expiry set to 15–60 minutes (access token); refresh token rotation enforced | `services/client-api/src/client_api/core/security.py` reviewed | ☐ |
| Refresh token revocation (logout) removes token from Redis blacklist | `services/client-api/src/client_api/api/v1/auth.py` logout endpoint reviewed | ☐ |
| OAuth CSRF state parameter validated in callback (Google OAuth flow) | `services/client-api/src/client_api/api/v1/auth.py` OAuth callback reviewed | ☐ |
| Admin JWT requires `role: admin` claim (not just a valid JWT) | `services/admin-api/src/admin_api/core/` admin auth dependency reviewed | ☐ |

### A08 — Software and Data Integrity Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| GitHub Actions CI pipeline enforces signed commits or branch protection rules | `.github/workflows/` CI config reviewed | ☐ |
| Celery task payloads are typed Pydantic models (no raw dict deserialization from untrusted sources) | `services/notification/src/notification/tasks/` reviewed | ☐ |

### A09 — Security Logging and Monitoring Failures

| Check | Evidence | Status |
|:---|:---|:---:|
| Audit log middleware records all write operations (POST/PUT/PATCH/DELETE) with user_id, action, entity | `services/shared/audit_middleware.py` reviewed (Story 2.11) | ☐ |
| Admin tier-override actions create audit log entries with reason field | `services/admin-api/src/admin_api/api/v1/tenants.py` reviewed | ☐ |
| Failed auth attempts logged (login failure, invalid API key) | `services/client-api/src/client_api/api/v1/auth.py` + enterprise-api middleware reviewed | ☐ |

### A10 — Server-Side Request Forgery (SSRF)

| Check | Evidence | Status |
|:---|:---|:---:|
| No user-supplied URLs fetched server-side (report URL, webhook, redirect) without validation | All API endpoints reviewed; no user-controlled URL fetch patterns found | ☐ |
| AI gateway outbound requests are to fixed LLM provider endpoints (not user-controlled) | `services/ai-gateway/` config reviewed | ☐ |

---

## Network Policy Verification

Run the following in the staging cluster after deploying network policy changes (AC4):

```bash
# Verify all 6 services have network policies
kubectl get networkpolicy -n eusolicit

# Confirm client-api cannot reach admin-api directly (should timeout/refuse)
kubectl exec -n eusolicit deployment/client-api -- \
  curl -m 5 http://admin-api:8002/healthz

# Confirm enterprise-api can reach client-api proxy
kubectl exec -n eusolicit deployment/enterprise-api -- \
  curl -m 5 http://client-api:8001/healthz

# Confirm notification cannot be reached from outside the cluster
# (no ingress rules — all inbound traffic should be blocked)
```

| Test | Expected | Actual | Pass? |
|:---|:---|:---|:---:|
| `client-api → admin-api` direct call | Connection refused / timeout | | ☐ |
| `enterprise-api → client-api` proxy | 200 OK | | ☐ |
| `notification` inbound (no ingress) | Connection refused | | ☐ |

---

## Secret Rotation Runbook

All production secrets must be rotated before the release gate. Document the completion of each rotation below.

| Secret | Location | Rotated By | Date | New Value Stored In | Verified? |
|:---|:---|:---|:---|:---|:---:|
| `CLIENT_API_RSA_PRIVATE_KEY` (JWT signing key) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_RSA_PUBLIC_KEY` (JWT verification key) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `ADMIN_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `admin-api-secrets` | | | AWS Secrets Manager | ☐ |
| `ENTERPRISE_API_DATABASE_URL` (PostgreSQL password) | K8s Secret `enterprise-api-secrets` | | | AWS Secrets Manager | ☐ |
| `NOTIFICATION_DATABASE_URL` (PostgreSQL password) | K8s Secret `notification-secrets` | | | AWS Secrets Manager | ☐ |
| `INTERNAL_SERVICE_SECRET` (service-to-service auth) | K8s Secret (all services) | | | AWS Secrets Manager | ☐ |
| `CLIENT_API_OAUTH_SECRET_KEY` (session cookie signing) | K8s Secret `client-api-secrets` | | | AWS Secrets Manager | ☐ |
| `SENDGRID_API_KEY` | K8s Secret `notification-secrets` | | | AWS Secrets Manager | ☐ |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` (S3) | K8s Secret (multiple services) | | | AWS IAM (role rotation) | ☐ |

> **Rotation procedure:**
> 1. Generate new secret value (use `openssl rand -base64 48` for symmetric keys; `openssl genrsa -out key.pem 4096` for RSA pairs).
> 2. Store in AWS Secrets Manager under the path `eusolicit/production/<secret-name>`.
> 3. Update the Kubernetes Secret via External Secrets Operator (or `kubectl create secret generic … --dry-run=client -o yaml | kubectl apply -f -`).
> 4. Perform a rolling restart: `kubectl rollout restart deployment/<service-name> -n eusolicit`.
> 5. Verify service health: `kubectl rollout status deployment/<service-name> -n eusolicit`.
> 6. Mark the row above as verified.

---

## Sign-Off

| Role | Name | Signature | Date |
|:---|:---|:---|:---|
| Security Lead | | | |
| Tech Lead | | | |
| Product Owner | | | |
