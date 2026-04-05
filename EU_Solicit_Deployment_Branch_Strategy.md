# EU Solicit — Deployment & Branch Strategy

**Version**: 1.0 | **Date**: 2026-04-05 | **Status**: Proposal | **Ref**: Solution Architecture v3.1

---

## 1. Repository Structure

**Decision**: Monorepo with per-service independence.

The architecture defines 5 services, 3 shared libraries, 1 frontend, and infrastructure-as-code — all in a single repository. This aligns with the 2-3 person team size and the shared library dependencies (`eusolicit-common`, `eusolicit-kraftdata`, `eusolicit-models`).

```
eusolicit/
├── services/
│   ├── client-api/
│   ├── admin-api/
│   ├── data-pipeline/
│   ├── ai-gateway/
│   └── notification/
├── packages/
│   ├── eusolicit-common/
│   ├── eusolicit-kraftdata/
│   └── eusolicit-models/
├── frontend/
│   ├── client/            # eusolicit.com
│   └── admin/             # admin.eusolicit.com
├── infra/
│   ├── terraform/
│   ├── helm/
│   │   ├── base-chart/    # Reusable Helm template
│   │   └── values/        # Per-service + per-env overrides
│   └── docker-compose/    # Local dev
├── scripts/
├── .github/workflows/
└── docs/
```

**Rationale**: With a small team, a monorepo reduces coordination overhead — shared library changes, cross-service refactors, and infrastructure updates land in a single PR. GitHub Actions path filters ensure only affected services are built and tested.

---

## 2. Branch Strategy — Trunk-Based Development

**Decision**: Trunk-based development with short-lived feature branches. No long-lived `develop` or `release/*` branches.

### 2.1 Branch Model

| Branch | Purpose | Lifetime | Protection |
|---|---|---|---|
| `main` | Production-ready trunk | Permanent | Required reviews, status checks, no force push |
| `feature/<ticket>-<slug>` | New features and enhancements | 1-3 days max | PR required to merge into `main` |
| `fix/<ticket>-<slug>` | Bug fixes | < 1 day | PR required |
| `hotfix/<ticket>-<slug>` | Production emergency fixes | Hours | PR required, expedited review |
| `infra/<slug>` | Terraform, Helm, CI/CD changes | 1-3 days | PR required |

### 2.2 Why Trunk-Based (Not GitFlow)

GitFlow adds unnecessary ceremony for a 2-3 person team shipping a SaaS product. The architecture already isolates blast radius at the service level — that's where safety lives, not in long-lived branches. Trunk-based gives you continuous integration without merge hell, fast feedback via CI on every push, simpler mental model (one source of truth), and natural alignment with the per-service independent deployment model from v3.

### 2.3 Branch Naming Convention

```
feature/EUS-42-opportunity-search
fix/EUS-87-stripe-webhook-retry
hotfix/EUS-103-jwt-validation-bypass
infra/upgrade-redis-helm-chart
```

### 2.4 Merge Rules

All merges to `main` use **squash merge**. This keeps the trunk linear, makes reverts trivial, and ensures each commit on `main` maps to exactly one PR.

---

## 3. Environments

Three environments, matching the architecture's environment strategy (Section 14.3).

| Environment | Trigger | Infra | URL | KraftData |
|---|---|---|---|---|
| **Local** | `docker compose up` | Docker Compose | `localhost:*` | KraftData stage |
| **Staging** | Auto-deploy on `main` merge | K8s (small cluster, eu-central-1) | `stage.eusolicit.com` | KraftData stage |
| **Production** | Manual promotion from staging (tagged release) | K8s (HA cluster, eu-central-1) | `eusolicit.com` | KraftData production |

### 3.1 Environment Parity

All environments run the same Docker images, Helm charts, and database migrations. Differences are isolated to Helm values files:

```
infra/helm/values/
├── local.yaml
├── staging.yaml
└── production.yaml
```

Secrets are injected via External Secrets Operator (staging/prod) or `.env` files (local).

---

## 4. CI/CD Pipeline — GitHub Actions

### 4.1 Change Detection

Every pipeline begins with path-based change detection. A change to `services/client-api/` triggers only the client-api pipeline. A change to `packages/eusolicit-common/` triggers **all** dependent services.

```yaml
# .github/workflows/ci.yaml — path filter matrix
paths:
  client-api:    ['services/client-api/**', 'packages/**']
  admin-api:     ['services/admin-api/**', 'packages/**']
  data-pipeline: ['services/data-pipeline/**', 'packages/**']
  ai-gateway:    ['services/ai-gateway/**', 'packages/**']
  notification:  ['services/notification/**', 'packages/**']
  frontend:      ['frontend/**']
  infra:         ['infra/**']
```

### 4.2 PR Pipeline (on every push to feature/fix/hotfix branches)

```
┌─────────────────────────────────────────────────────────────────┐
│  PR Opened / Updated                                            │
│                                                                 │
│  1. Change Detection (which services changed?)                  │
│                                                                 │
│  2. Parallel per-service matrix:                                │
│     ├── Lint (ruff)                                             │
│     ├── Type check (mypy)                                       │
│     ├── Unit tests (pytest)                                     │
│     └── Security scan (bandit + pip-audit)                      │
│                                                                 │
│  3. Frontend (if changed):                                      │
│     ├── Lint (eslint) + Type check (tsc)                        │
│     └── Unit tests (vitest)                                     │
│                                                                 │
│  4. Shared libraries (if changed):                              │
│     └── Unit tests for packages/*                               │
│                                                                 │
│  5. Infra validation (if changed):                              │
│     ├── Terraform fmt + validate                                │
│     └── Helm lint + template render                             │
│                                                                 │
│  ──► All green = PR is mergeable                                │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Merge-to-Main Pipeline (staging deploy)

```
┌─────────────────────────────────────────────────────────────────┐
│  PR Merged to main                                              │
│                                                                 │
│  1. Change Detection                                            │
│                                                                 │
│  2. Integration tests (Docker Compose, all 5 services)          │
│     └── API contract tests, cross-service event tests           │
│                                                                 │
│  3. Build Docker images (parallel matrix, affected only)        │
│     └── Tag: sha-<commit>, branch-main                          │
│                                                                 │
│  4. Push to container registry (ECR eu-central-1)               │
│                                                                 │
│  5. Deploy to staging                                           │
│     ├── Alembic migrations (per-service, ordered)               │
│     ├── Helm upgrade (affected services only)                   │
│     └── Wait for rollout                                        │
│                                                                 │
│  6. Smoke tests against stage.eusolicit.com                     │
│     ├── Health checks (all services)                            │
│     ├── Auth flow (registration, login, JWT)                    │
│     ├── Opportunity search (Client API → Pipeline schema)       │
│     └── AI Gateway circuit breaker (KraftData stage)            │
│                                                                 │
│  ──► Staging is always up-to-date with main                     │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Production Release Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  Git Tag: v*.*.*                                                │
│                                                                 │
│  1. Build + push Docker images tagged: v1.2.3                   │
│                                                                 │
│  2. Deploy to staging (verify tag works in staging)             │
│     └── Full smoke test suite                                   │
│                                                                 │
│  3. ⏸  Manual approval gate (GitHub Environment protection)     │
│                                                                 │
│  4. Deploy to production                                        │
│     ├── Alembic migrations (per-service, ordered)               │
│     ├── Helm upgrade with --atomic (auto-rollback on failure)   │
│     └── Progressive rollout (25% → 50% → 100%)                 │
│                                                                 │
│  5. Post-deploy verification                                    │
│     ├── Health checks                                           │
│     ├── Smoke tests against eusolicit.com                       │
│     └── Grafana alert silence window (5 min, watch for spikes)  │
│                                                                 │
│  6. GitHub Release created automatically                        │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 Hotfix Flow

```
main ──────────────────────────────────────────►
         \                         /
          hotfix/EUS-103-fix ─────►  (PR, squash merge)
                                     │
                                     ▼
                                   Tag v1.2.4
                                     │
                                     ▼
                              Deploy staging → approval → production
```

Hotfixes branch from `main`, get expedited review (single reviewer), and follow the same tag → staging → approval → production flow. No shortcuts on the pipeline — even hotfixes run through staging first.

---

## 5. Versioning Strategy

**Decision**: Single semver version for the platform, with per-service build metadata.

```
Platform version:   v1.2.3
Docker image tags:  eusolicit/client-api:v1.2.3
                    eusolicit/admin-api:v1.2.3
                    eusolicit/ai-gateway:v1.2.3
                    ...
```

### 5.1 Why Unified Versioning (Not Per-Service)

Per-service versioning makes sense at scale (50+ engineers, independent release cadences). For a 2-3 person team, it introduces version matrix complexity with no real benefit. A single version means every release is a known-good combination of all services, rollbacks are atomic (revert the entire platform to v1.2.2), and it's simpler to communicate to stakeholders and in audit logs.

### 5.2 Semver Rules

| Change Type | Version Bump | Example |
|---|---|---|
| Breaking API change (Enterprise API) | Major | v2.0.0 |
| New feature, non-breaking | Minor | v1.3.0 |
| Bug fix, performance, dependency update | Patch | v1.2.4 |

---

## 6. Database Migration Strategy

Each service owns its schema and runs Alembic migrations independently. Migrations run **before** the new service version starts.

### 6.1 Migration Execution Order

```
1. shared       (depended on by all)
2. pipeline     (depended on by client-api reads)
3. gateway      (independent)
4. notification (reads from client, pipeline)
5. admin        (reads from client, pipeline)
6. client       (reads from pipeline)
```

### 6.2 Migration Safety Rules

- **No destructive migrations in production** — column drops happen in two releases (release N: stop using column, release N+1: drop column).
- **Backward-compatible only** — new columns must have defaults or be nullable. Old code must still work against the new schema during rolling deploys.
- **Migration testing** — integration tests apply all migrations against a fresh DB to catch ordering issues.
- **Migration lock** — use `pg_advisory_lock` in the Alembic env to prevent concurrent migration runs.

---

## 7. Deployment Mechanics

### 7.1 Helm Chart Structure

A single base Helm chart template shared across all services, with per-service values overrides:

```
infra/helm/
├── base-chart/
│   ├── Chart.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── hpa.yaml
│       ├── networkpolicy.yaml
│       ├── ingress.yaml          # Optional, not all services
│       └── externalsecret.yaml
└── values/
    ├── client-api/
    │   ├── base.yaml             # Service-specific defaults
    │   ├── staging.yaml          # Staging overrides
    │   └── production.yaml       # Production overrides
    ├── admin-api/
    │   └── ...
    └── ...
```

### 7.2 Rolling Deployment Configuration

```yaml
# Default for all services
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0           # Zero-downtime

# Celery Beat (singleton) — Recreate strategy
# Only data-pipeline-beat and notification-beat
strategy:
  type: Recreate               # Prevent duplicate schedulers
```

### 7.3 Health Checks

Every service exposes `/health/live` (liveness) and `/health/ready` (readiness). Readiness checks verify DB connectivity, Redis connectivity, and (for AI Gateway) KraftData reachability.

### 7.4 Rollback Procedure

```bash
# Automatic: Helm --atomic rolls back on failed deploy
# Manual: revert to previous release
helm rollback eu-solicit-client-api <revision> -n eu-solicit

# Nuclear option: revert entire platform
for svc in client-api admin-api ai-gateway data-pipeline notification frontend; do
  helm rollback eu-solicit-$svc <revision> -n eu-solicit
done
```

Database rollbacks require running Alembic downgrade scripts, which are tested as part of the migration test suite.

---

## 8. Release Cadence

| Phase | Cadence | Rationale |
|---|---|---|
| MVP Development (Weeks 1-20) | Deploy to staging on every merge, production releases weekly or as features complete | Fast iteration, frequent feedback |
| Post-MVP / Growth | Deploy to staging on every merge, production releases bi-weekly | Stability with regular feature delivery |
| Mature / Enterprise | Same as above + release notes, API changelog, deprecation notices | Enterprise API contract obligations |

---

## 9. Infrastructure as Code

### 9.1 Terraform Structure

```
infra/terraform/
├── modules/
│   ├── eks/                  # K8s cluster
│   ├── rds/                  # PostgreSQL (or K8s StatefulSet for MVP)
│   ├── elasticache/          # Redis
│   ├── s3/                   # Document storage
│   ├── ecr/                  # Container registry
│   ├── secrets-manager/      # Per-service secrets
│   └── cloudflare/           # DNS + WAF
├── environments/
│   ├── staging/
│   │   └── main.tf           # Composes modules
│   └── production/
│       └── main.tf
└── backend.tf                # S3 + DynamoDB state locking
```

### 9.2 Terraform CI

Terraform changes run `plan` on PR, `apply` on merge to `main` (staging) or on tag (production), both gated by manual approval.

---

## 10. Security Controls in the Pipeline

| Control | Where | Tool |
|---|---|---|
| Dependency vulnerability scan | PR pipeline | `pip-audit`, `npm audit` |
| Static analysis | PR pipeline | `bandit` (Python), `eslint-plugin-security` |
| Container image scan | Post-build | Trivy or ECR native scanning |
| Secret detection | Pre-commit + PR | `gitleaks` |
| SBOM generation | Release pipeline | `syft` → attach to GitHub Release |
| Network policy verification | Staging smoke tests | `kubectl` assertions on NetworkPolicy |

---

## 11. Local Development Workflow

```bash
# Full stack (all 5 services + dependencies)
docker compose -f infra/docker-compose/docker-compose.yaml up

# Single service development (run one service natively, rest in Docker)
docker compose up postgres redis minio clamav
cd services/client-api && uvicorn src.infrastructure.api.app:app --reload

# Run tests for a single service
cd services/client-api && pytest

# Run all service tests (mirrors CI matrix)
./scripts/test-all.sh
```

---

## 12. Decision Summary

| Decision | Choice | Key Rationale |
|---|---|---|
| Repository | Monorepo | Small team, shared libraries, atomic cross-service changes |
| Branch model | Trunk-based, short-lived branches | Fast integration, low ceremony for 2-3 devs |
| Merge strategy | Squash merge | Linear history, clean reverts |
| Versioning | Unified semver | Known-good combinations, simple rollbacks |
| Staging deploy | Auto on merge to `main` | Continuous integration, always-fresh staging |
| Production deploy | Manual approval on semver tag | Safety gate, auditability |
| Migrations | Per-service Alembic, ordered execution | Schema isolation from v3 architecture |
| Rollback | Helm `--atomic` + manual rollback | Zero-downtime, safe production |
| IaC | Terraform with plan-on-PR, apply-on-merge | Peer-reviewed infra changes |

---

*Document version: 1.0 | Date: 2026-04-05 | Status: Proposal*
