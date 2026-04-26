# E18: Trust Center & Compliance Posture

**Sprint**: 14–15 | **Points**: 13 | **Dependencies**: E03 (Frontend Shell), E07 (WeasyPrint pipeline), E09 (notification service) | **Milestone**: M2 hard deadline
**Source:** PRD v1.1 §6 FR10 + §8 US10; architecture-evaluation §2 Change-4, §11 locked decision §11.4 (M2 deadline)

## Goal

Ship the public Trust Center page (`/trust`) by **Month 2 of platform launch** — a hard deadline locked because mid-large EU consulting-firm deals stall 4–8 weeks at the procurement gate without downloadable security/compliance evidence. ISO 27001 audit-prep is a parallel sprint-spanning programme (M2–M12, €50K budget) that this epic enables but does not deliver. Trust Center renders statically from version-controlled MDX + YAML so every content change is auditable in Git history (which is what ISO 27001 wants for change management). PDF artefacts split: human-curated for legal (DPA, BCP), generated via WeasyPrint pipeline (Epic 7) for living artefacts (sub-processor list, security overview).

## Acceptance Criteria

- [ ] Public route `/trust` accessible without authentication; ingress rule allows `/trust/*` without JWT
- [ ] Page lists current compliance posture: GDPR (compliant), ISO 27001 (in progress with target date M12), SOC 2 (planned/N-A — secondary in EU), data residency (EU only), encryption at rest (AES-256), in transit (TLS 1.3)
- [ ] Downloadable PDF artefacts: GDPR DPA, Security Overview, Sub-Processor List, latest Pen-Test Summary (redacted), BCP Summary, Data Residency Confirmation
- [ ] Sub-Processor List auto-generated from `infra/sub-processors.yaml` via WeasyPrint pipeline; Git commits trigger re-render in CI
- [ ] All Trust Center artefacts versioned with change-log entries visible to subscribers
- [ ] Sub-processor changes auto-detected via Git diff in CI → emits `subprocessor.changed` event → notification service emails active customer DPAs (per GDPR Art. 28)
- [ ] ISO 27001 roadmap displayed with quarterly milestones (M3 Stage 1 audit / M9–10 Stage 2 audit / M12 certification awarded)
- [ ] M2 deadline tracked in `eusolicit-docs/` with progress visible in sprint review
- [ ] No authenticated routes leak from `/trust` namespace; cross-tenant negative test: unauthenticated request returns full content with no user data

## Stories

### S18.00: Public /trust Route + MDX Pipeline + Sub-Processor YAML + Change-Log Generator
**Points**: 5 | **Type**: frontend

Frontend (`apps/client/`):
- New route `apps/client/app/(public)/trust/page.tsx` — public segment, no auth wrapper, server-side rendered
- Source content from `apps/client/content/trust/*.mdx` and `apps/client/content/trust/sub-processors.yaml`
- Page renders compliance-posture summary, certification roadmap (with current stage indicator), and downloadable artefact links
- Layout reuses existing `<AppShell>` patterns minus authenticated topbar (public variant)
- BG and EN locales; i18n key parity check passes

Ingress configuration:
- nginx ingress route allows `/trust/*` without JWT verification (config update in Helm chart)
- Cloudflare WAF rule confirms public access without auth challenge

Sub-Processor YAML schema (`infra/sub-processors.yaml`):
```yaml
sub_processors:
  - name: KraftData (Sirma AI)
    purpose: AI agent execution + vector storage
    region: EU (specific datacentre TBD)
    effective_date: 2026-04-01
  - name: Stripe
    purpose: Payment processing
    region: EU + Global (per Stripe DPA)
    effective_date: 2026-04-01
  # ... etc
```

Change-log generator:
- CI step on every commit that modifies `infra/sub-processors.yaml`: computes diff vs previous version, appends entry to `infra/sub-processors-changelog.md` with timestamp, additions, removals
- Entry visible on Trust Center page in change-log section

Tests:
- Cross-tenant negative test: unauthenticated request to `/trust` returns full content, no user-specific data leaks
- i18n parity test (BG/EN match)
- ATDD source-inspection: page uses Next.js MDX pipeline (NOT runtime Markdown rendering — security/auditability)

---

### S18.01: PDF Artefact Pipeline (WeasyPrint Reuse) for Living Artefacts; Git-Managed for Legal Artefacts
**Points**: 5 | **Type**: backend + content

PDF generation pipeline (reuses Epic 7 WeasyPrint patterns; CRITICAL: per project-context Epic 7 pattern, "ThreadPoolExecutor is mandatory for CPU-bound library calls"):
- `services/integrations-api/` (or extend existing utility — defer to architect) exposes `POST /api/internal/trust/render-pdf` (admin-only, internal)
- Renders MDX files to PDF on-demand or in CI nightly
- Output stored in S3 with versioning enabled
- Signed URLs (1-hour TTL) returned to frontend for download

**Living artefacts (auto-generated):**
- Security Overview (from `apps/client/content/trust/security-overview.mdx`)
- Sub-Processor List (from `infra/sub-processors.yaml`)
- Data Residency Confirmation (from `apps/client/content/trust/data-residency.mdx`)
- BCP Summary (from `apps/client/content/trust/bcp-summary.mdx`)

**Legal artefacts (Git-managed PDFs):**
- GDPR Data Processing Agreement — `infra/trust/artefacts/dpa/v1.pdf` (legal-curated)
- Latest Pen-Test Summary (redacted) — `infra/trust/artefacts/pen-test/2026-Q4.pdf` (auditor-curated)

Frontend route serves both types via the same download link pattern; live PDFs re-rendered on content change, legal PDFs served as-is from version-controlled storage.

Tests:
- WeasyPrint runs in `run_in_executor` (project-context Epic 7 pattern — verified by source-inspection ATDD)
- Sub-processor PDF reflects current `sub-processors.yaml` content (regression test)
- Signed URL expiry behaviour (1-hour TTL test)

---

### S18.02: Sub-Processor Change → DPA Notification Flow (Notification Service Extension)
**Points**: 3 | **Type**: backend

CI integration:
- GitHub Actions step on every push to main that modifies `infra/sub-processors.yaml`: computes diff, publishes `subprocessor.changed` event to Redis Streams (`eu-solicit:notifications` stream) with payload `{added: [...], removed: [...], effective_date: ...}`

Notification service extension:
- Subscribe to `subprocessor.changed` events
- Identify all active customer DPAs (companies on paid tiers from `client.companies` + `client.subscriptions`)
- Send email via SendGrid using new `subprocessor_change` template (BG/EN); template includes added/removed sub-processors and effective date
- Log delivery to `notification.email_log` per existing pattern
- Use canonical fire-and-forget audit write pattern (project-context Epic 13 Rule 45) — failure logs but does not block

GDPR Art. 28 compliance:
- 30-day advance notification for new sub-processors (effective_date in YAML must be ≥30 days future for adds)
- CI lint step: rejects YAML changes where new sub-processor `effective_date` is <30 days future
- Customer right-to-object documented in email body with contact link

Tests:
- Event-driven test: simulate `subprocessor.changed` event → all paid customers receive email within 60s
- 30-day-future lint test: PR adding sub-processor with near-term effective_date fails CI
- i18n parity for email template (BG/EN match)
- Audit log entry written for each delivery
