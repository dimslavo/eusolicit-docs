# Story 21.4: PodDisruptionBudgets + Min-Replica Enforcement Across All Services

Status: review

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream guidance — non-negotiable.
     AP18-C2: orchestrator MUST patch `Status:` in this file atomically with the sprint-status transition.
     AP17-C1 two-gate-close: `Status: done` requires bmad-code-review **Approve** verdict (Pass-2). Dev pass alone does NOT promote to done — protect the S19-0/19-1/19-2/20-0/21-1/21-2/21-3 successful-closure streak (7 in a row, with Story 21-3 the most recent dev-pass + review-fixpass; 21-3 Approve verdict pending at create-time).
     Operator workflow guidance for E21 (multi-story epic):
       - [IR] already executed 2026-05-04 v1+v2 + 2026-05-05 for E21. DO NOT re-run.
       - [VS] Validate Story BEFORE bmad-dev-story — non-negotiable.
       - [SR] Story Review AFTER this story closes (multi-story epic; PE.05 SLO dashboards consume the new HPA / PDB / chaos-drill metrics; PE.06 runbooks reference this story's chaos-drill procedure).
       - [PR] Post-Review after code review is complete (mandated for ALL epics).
       - [ER] Epic Review at end of E21 (interdependent stories: PE.01 → PE.02 → PE.03 → PE.04 → PE.05 → PE.06 chain).
       - epic-21 status remains in-progress (transitioned on Story 21-1 create). -->

## Story

As a **Platform Engineering lead** (with the CTO as the executive sponsor of the publishable 99.9% SLA),
I want **(1) every production service Helm chart updated with a `PodDisruptionBudget` (PDB) resource set to `minAvailable: 1` so that voluntary disruptions — node drains during cluster maintenance, EKS managed-node-group rolling upgrades, cluster-autoscaler scale-down events, kubectl drain during chaos drills, the kubelet-eviction path during memory pressure — can never take all replicas of a single service down at once; (2) production `minReplicas` (HPA `minReplicas` for the autoscaling-enabled deployments) raised to the per-service floors mandated by epic spec line 105 — `client-api=3` (already at 3 ✓), `admin-api=2` (currently HPA `minReplicas: 1` ✗ — bump to 2), `ai-gateway=2` (already at 2 ✓), `data-pipeline=2` (already at 2 ✓ — note: epic line 105 calls this "data-pipeline-worker"; in our Helm topology the data-pipeline service runs both the FastAPI HTTP API and the Celery workers from the same image — the `minReplicas: 2` floor applies to the unified deployment), `notification=2` (currently HPA `minReplicas: 1` ✗ — bump to 2), and `integrations-api=2` (NO Helm values file exists yet — must be CREATED per ADR-009 line 743 which establishes integrations-api as a separate service on port 8007); (3) the PDB+min-replica pattern wired through the existing `infra/helm/eusolicit-service/templates/pdb.yaml` template — the template already supports `podDisruptionBudget.enabled` + `podDisruptionBudget.minAvailable` from Story 1.9 (line 1 conditional `{{- if .Values.podDisruptionBudget.enabled }}`), so the change is value-file-side only for the 5 existing values files plus a NEW values file for integrations-api; (4) HPA `maxReplicas` ceilings reviewed against the Story 21-1 §Sizing Recommendations for PE.04 evidence (load-test-results.md lines 808–827) — the AI-Gateway PDB `minAvailable: 1` + HPA `minReplicas: 2` recommendation stands per line 817 ("default PDB `minAvailable: 1` and HPA min-replicas: 2 stand"); the data-pipeline queue-depth-driven scale decision is deferred per line 826 ("queue-depth-driven HPA scale decision deferred to staging measurement") — D-3 below pre-records this as a deferrable deviation; (5) the nginx-ingress controller's PDB verified — nginx-ingress runs as a separate Helm release (`ingress-nginx/ingress-nginx` chart) with its own values; the upstream chart already ships `controller.minAvailable=1` + `controller.replicaCount=2` defaults but our deployment may not have set them, so the verification step is `helm get values ingress-nginx -n ingress-nginx` + `kubectl get pdb -n ingress-nginx` against staging and a documented values-override patch if the PDB or 2-replica minimum is missing; (6) NetworkPolicies inspected to confirm HA failover paths — specifically, every cross-service egress rule must use `podSelector` (label-matching) NOT pod-IP literals, so that when a replica fails over to a new pod IP the policy continues to allow the connection (already true today — `client-api.yaml` networkPolicy.egress to `ai-gateway` uses `podSelector: matchLabels: app.kubernetes.io/name: ai-gateway` per lines 113–120 — this AC is a verification + lint step, not new code); (7) cluster-autoscaler configured with the right scale-out behaviour to handle replica scale-out under load — capture the existing managed-node-group `min_size`/`max_size`/`desired_capacity` Terraform settings (out-of-tree to this repo per ADR-010 — staging+prod EKS clusters are SRE-provisioned at the AWS account level, not from this repo's `infra/terraform/`) and document them in the new chaos-drill runbook so the chaos test can verify scale-out works; (8) a staged **chaos drill** — `kubectl drain` (or `kubectl cordon`+`kubectl delete pod`) a node hosting at least one replica of each of the 6 services; verify (a) PDB rejects the drain if it would violate `minAvailable: 1`, (b) the surviving replicas continue serving traffic, (c) Kubernetes evicts the pod within the PDB constraint (typically by waiting for a new replica to come up first), (d) the drained pod re-schedules onto a different node, (e) the new pod becomes Ready within 60 seconds, (f) zero 5xx errors are observed on the service's external endpoint during the drain (measured via a parallel `curl --max-time 3` polling loop at 1-second intervals against `/healthz`); (9) a NEW Helm-chart-lint CI step in `eusolicit-app/.github/workflows/ci.yml` (or `quality-gates.yml`) that runs `python scripts/check_helm_pdb_and_minreplicas.py` (NEW script) and **fails the build** if any per-service Helm values file under `infra/helm/values/*.yaml` lacks (a) `podDisruptionBudget.enabled: true` for production, OR (b) `podDisruptionBudget.minAvailable: 1` (or `maxUnavailable` semantically equivalent — but `minAvailable` is the project convention), OR (c) `autoscaling.minReplicas` below the per-service floor declared in `scripts/pe04_min_replica_floors.py` (NEW data file: client-api=3, admin-api=2, ai-gateway=2, data-pipeline=2, notification=2, integrations-api=2) — this is the **regression gate** that prevents future services from being added without HA primitives (epic line 112: "Helm-chart-lint CI step rejects future services without PDB"); (10) a NEW chaos-drill evidence file at `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-runbook.md` mirroring the PE.02/PE.03 cutover-runbook structure (10 sections: §Pre-flight + §Drain Procedure + §Per-Service Drill + §Validation + §Rollback + §PDB-Behaviour Evidence + §Cluster-Autoscaler Behaviour + §Network-Policy Verification + §Chaos-Drill Results table + §Sign-off),**
so that **(a) the 99.9% SLA promised in PRD v1.1 §7 NFR-14 has the Kubernetes-orchestration reliability foundation it requires (no voluntary disruption can take a service to zero replicas; AZ failure of one EKS managed-node-group AZ leaves at least 1 replica of every service alive; cluster-upgrade rolling drains complete safely; kubelet eviction under memory pressure honours the PDB); (b) the canonical Epic 21 SLA-publication gate `PE.01 + PE.02 + PE.03 + PE.04` advances from 3/4 (PE.01 done + PE.02 done + PE.03 dev-pass-pending-Approve) to 4/4 (PE.04 done) — which means the public 99.9% SLA announcement is unblocked from a code-and-config standpoint (PE.05 SLO dashboards + PE.06 on-call rotation are parallel/non-gating per epic line 19's "milestone" definition + epic line 20's gate definition "PE.01 + PE.02 + PE.03 + PE.04" — note: the SLA gate text reads "PE.01 + PE.02 + PE.03 + PE.04 ship" which this story closes; PE.05/PE.06 strengthen but do not gate); (c) PE.05 SLO dashboards (per E21 epic line 122) consume the new HPA + PDB + chaos-drill Prometheus metrics — `kube_poddisruptionbudget_status_current_healthy`, `kube_horizontalpodautoscaler_status_current_replicas`, `kube_pod_container_status_restarts_total` — that this story's metric annotations preserve; (d) PE.06 runbooks (epic line 138 — "Top-10 runbooks ... ingress controller restart, full-disk on PG ... deploy rollback") reference this story's `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill as the canonical procedure for safely removing a node; (e) the ADR-010 "boring-tech wins (Winston principle)" decision is honoured — managed EKS handles node-level HA via PDB-aware drain semantics + managed-node-group rolling-upgrade orchestration + Cluster Autoscaler reschedule logic, all of which require that PDBs are correctly authored on the workload side; (f) future services (e.g. eventual SSO service, billing-webhook receiver, etc.) cannot be added to the platform without the lint gate forcing them to author a PDB and meet the min-replica floor — this is the project's **HA-by-default** invariant; (g) the chaos-drill evidence file becomes the canonical regression-test fixture: any future Helm-template change that breaks the PDB+min-replica pattern shows up as a diff against this file's recorded observations.**

## Epic Context

- **Epic**: E21 Platform Reliability for 99.9% SLA — Sprint 14–17 (parallel to E14–E20 feature work); 34 pts; **6-story platform-engineering epic**. Milestone: **99.9% SLA externally publishable** (gated on PE.01 + PE.02 + PE.03 + **PE.04**).
- **Story points**: 5 | **Type**: platform-engineering / Helm + chaos drill | **Position**: FOURTH PE story after PE.01 (k6 baseline) done, PE.02 (PG HA) done, PE.03 (Redis HA) dev-pass-pending-Approve.
- **NFRs covered**: **NFR-14** (99.9% uptime SLA — PDBs + min-replicas are the workload-orchestration reliability primitive complementing PE.02/PE.03's data-tier primitives — without PDBs a single voluntary node-drain can take a service from N to 0 replicas in <5 seconds, well beyond the 43-min/month error budget); **NFR-13** (10K active companies / 1M opportunities, <20% degradation — the HPA `minReplicas` floor ensures the service has enough capacity at the start of any traffic ramp; Story 21-1 §Sizing Recommendations for PE.04 line 817 confirms the `minReplicas: 2` floor for AI-Gateway is the correct default); **NFR-17** (DR RTO ≤4h — PDBs ensure node-level HA failover proceeds within the budget by guaranteeing a Ready replica is always running while drained pods reschedule).
- **Position in epic chain**: **Fourth PE story**. Hard-depends on Story 21-1 outputs: (a) `load-test-results.md` §Sizing Recommendations for PE.04 (lines 808–827) — the recommendation rationale ("default PDB `minAvailable: 1` and HPA min-replicas: 2 stand" per line 817; queue-depth scale decision deferred per line 826 — pre-recorded as D-3 below); (b) `tests/load/k6-perf-core-flows.js` (Story 21-1 deliverable — used in chaos drill §Validation step as traffic generator). Soft-depends on Story 21-2's Helm chart layout (`infra/helm/eusolicit-service/templates/`) — AC-3 below adds the new `integrations-api.yaml` values file to the same `values/` directory. Soft-depends on Story 21-3's `redis.bareKey` + `redis.celeryEnabled` Helm value flags — `integrations-api.yaml` AC-3 must set `celeryEnabled: false` (integrations-api does not run Celery) and `bareKey: false` (integrations-api Settings has env_prefix). Does NOT depend on PE.05 or PE.06 (those consume this story's outputs).
- **Source**: Epic spec `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 99–112 (PE.04 scope). Architecture `architecture.md` ADR-009 lines 743–746 (integrations-api as separate service on port 8007), ADR-010 lines 757–764 (boring-tech rationale — managed EKS over self-managed Kubernetes), §6.2 Production Topology, line 168 (integrations-api service entry — port 8007 + per-service DB role + new for v2.0), line 169 (enterprise-api as separate gateway — disambiguates from integrations-api), line 743 ADR-009 explicit per-service split. PRD v1.1 §7 NFR-14/13/17. Story 21-1 evidence: `load-test-results.md` lines 808–827 (§Sizing Recommendations for PE.04 — the pre-decision rationale audit trail). Story 1.9 evidence: `infra/helm/eusolicit-service/templates/pdb.yaml` (the existing PDB template — Story 21-4 only changes values, not the template; the template was authored by Story 1.9 with the conditional `{{- if .Values.podDisruptionBudget.enabled }}` already in place at line 1); `infra/helm/eusolicit-service/values.yaml` lines 95–97 (default `podDisruptionBudget.enabled: false` + `minAvailable: 1` — the per-service values overrides flip enabled to `true`). Story 1.9 helm template tests: `eusolicit-app/tests/unit/test_helm_templates.py` lines 12–22 (the AC-2 PDB conditional test pattern that AC-9 below extends). Test design fallback: no `test_artifacts/test-design-epic-21.md` exists (consistent with Story 21-1 D-7, Story 21-2 D-2, Story 21-3 D-7 deviations; this is acceptable for an infra+Helm-chart story whose quality gate is the populated chaos-drill evidence file + the new lint-gate CI step + the helm-template rendering tests).
- **Operator workflow guidance**: [IR] already executed 2026-05-04 v1+v2 + 2026-05-05 (do NOT re-run). `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` (autopilot) → `bmad-code-review` (Approve verdict required for `done` per AP17-C1) → `[SR] Story Review` (multi-story epic; PE.05/PE.06 read from this story's outputs) → `[PR] Post-Review` (mandatory for all epics) → `[ER] Epic Review` at end of E21 (interdependent stories).

## Acceptance Criteria

> Source-of-truth: epic spec lines 99–112 (PE.04 scope) + Story 21-1 §Sizing Recommendations for PE.04 (load-test-results.md lines 808–827). AC numbers below cover every epic line item plus carry-forward hardening from Story 21-1 (AI-Gateway sizing rationale), Story 21-2 (Helm chart layout, AP18-C2 atomic-patch rule, AP17-C1 two-gate-close protection), Story 21-3 (`redis.celeryEnabled` + `redis.bareKey` Helm value flags; integrations-api ESO pattern), Epic 12 retro (non-functional evidence-file rule), Story 1.9 (the `pdb.yaml` template), and the ADR-009 integrations-api separate-service invariant.

### AC-1 — PodDisruptionBudget (`minAvailable: 1`) on Every Production Service Helm Values File

**Given** the Story 1.9 base chart already ships the PDB template at `eusolicit-app/infra/helm/eusolicit-service/templates/pdb.yaml` (lines 1–18) gated by `{{- if .Values.podDisruptionBudget.enabled }}` AND the chart-level default at `infra/helm/eusolicit-service/values.yaml` lines 95–97 is `podDisruptionBudget.enabled: false` + `minAvailable: 1` (defaults are off so the local docker-compose dev loop is unaffected) AND the only existing per-service values file with PDB enabled today is `enterprise-api.yaml` (lines 137–139, set by Story 12.15) AND the 5 production-relevant per-service values files (`client-api.yaml`, `admin-api.yaml`, `ai-gateway.yaml`, `data-pipeline.yaml`, `notification.yaml`) currently inherit the chart default (PDB DISABLED — no PDB resource is rendered),

**When** the dev agent implements PE.04,

**Then**:
1. **Add `podDisruptionBudget` block** to **all 5** existing per-service values files (`client-api.yaml`, `admin-api.yaml`, `ai-gateway.yaml`, `data-pipeline.yaml`, `notification.yaml`) with the structure:
   ```yaml
   podDisruptionBudget:
     enabled: true
     minAvailable: 1
   ```
   Place the block at the same vertical position the existing `enterprise-api.yaml` does — between `networkPolicy:` and the `# External Secrets Operator` block — for visual consistency across all 6 values files.
2. **Verify `enterprise-api.yaml`** already has the block (lines 137–139). Do NOT modify it; leave the existing config in place. Do NOT remove the existing `serviceName: integrations-api` line — that is a pre-existing Secrets Manager naming convention from Story 21-2/21-3 (D-4 below pre-records this naming-disambiguation as out-of-scope for PE.04).
3. **Render gate**: `helm template eusolicit-<service> infra/helm/eusolicit-service/ -f infra/helm/values/<service>.yaml` MUST emit a `kind: PodDisruptionBudget` document with `spec.minAvailable: 1` AND `spec.selector.matchLabels` matching the deployment's selectorLabels (the template at `pdb.yaml` line 17 already does this via `{{- include "eusolicit-service.selectorLabels" . | nindent 6 }}` — verify the rendered selector is non-empty after the change).
4. **Anti-pattern guard #1**: NEVER set `podDisruptionBudget.maxUnavailable` instead of `minAvailable`. The template at `pdb.yaml` lines 9–14 supports both (mutually exclusive — Kubernetes rejects PDBs with both fields), but the project convention is `minAvailable: 1` because at `minReplicas: 2` it semantically maps to "at most 1 may be unavailable" — equivalent in this case but `minAvailable` is the convention the chaos-drill validation in AC-8 below queries.
5. **Anti-pattern guard #2**: NEVER set `minAvailable: 0` (the value `0` is technically allowed by Kubernetes — it disables the PDB — but defeats the purpose; the linter in AC-9 below treats `0` as a violation).
6. **Anti-pattern guard #3**: NEVER set `minAvailable` as a percentage (e.g. `"50%"`) without explicit reasoning — for our 2/3-replica floors the integer-form `1` is correct; percentage form re-introduces the math the linter would otherwise check (e.g. `50%` of 2 replicas = 1 replica, equivalent — but `33%` of 3 replicas rounds DOWN to 0 in some Kubernetes versions, breaking the invariant).

### AC-2 — Production `autoscaling.minReplicas` Floors per Epic Spec Line 105

**Given** the per-service `autoscaling.minReplicas` values are currently set in each `values/<service>.yaml` AND epic spec line 105 mandates `client-api=3, admin-api=2, ai-gateway=2, data-pipeline-worker=2, notification-worker=2, integrations-api=2`,

**When** the dev agent reviews and updates `autoscaling.minReplicas`,

**Then**:
1. **`client-api.yaml`**: leave `autoscaling.minReplicas: 3` unchanged (already at floor — line 29).
2. **`admin-api.yaml`**: change `autoscaling.minReplicas: 1` → `2` (line 29). admin-api is IP-allowlisted but still requires HA — a single-replica admin-api goes down during drain.
3. **`ai-gateway.yaml`**: leave `autoscaling.minReplicas: 2` unchanged (already at floor — line 29; rationale: Story 21-1 §Sizing Recommendations for PE.04 line 817).
4. **`data-pipeline.yaml`**: leave `autoscaling.minReplicas: 2` unchanged (already at floor — line 29; epic spec calls it "data-pipeline-worker" but in our Helm topology the data-pipeline service runs both the FastAPI HTTP API and the Celery workers from the same image — the `minReplicas: 2` floor applies to the unified deployment).
5. **`notification.yaml`**: change `autoscaling.minReplicas: 1` → `2` (line 29). notification is the Celery worker for outcome briefs + email + materialized-view refresh — losing the only replica during drain pauses all those flows.
6. **`integrations-api.yaml`** (NEW file per AC-3 below): set `autoscaling.minReplicas: 2`.
7. **`enterprise-api.yaml`**: leave `autoscaling.minReplicas: 2` unchanged (line 29). Story 12.15 already set this; PE.04 verifies but does not change.
8. **HPA `maxReplicas` review**: AC requires reading the existing `maxReplicas` against Story 21-1 §Sizing Recommendations and recording an §HPA Sizing Review row in the chaos-drill runbook (AC-10) for each service. Existing values: client-api=20, admin-api=3, ai-gateway=10, data-pipeline=8, notification=4, integrations-api=10 (NEW), enterprise-api=10. The Story 21-1 evidence says "queue-depth-driven HPA scale decision deferred to staging measurement" (line 826) — D-3 below pre-records this as a deferrable deviation; **maxReplicas values are NOT changed in this story** unless a specific Story 21-1 measurement in load-test-results.md directly contradicts the current setting.
9. **Anti-pattern guard #4**: NEVER set `autoscaling.minReplicas: 1` on any service whose PDB has `minAvailable: 1` — this is the **dead-lock case**: PDB rejects every drain attempt because evicting the only replica would violate `minAvailable: 1`, which means cluster maintenance + node upgrades cannot complete and EKS managed-node-group rolling upgrades stall. The lint gate in AC-9 below treats `(minReplicas == 1 AND PDB.minAvailable >= 1)` as a HARD FAIL.
10. **Anti-pattern guard #5**: NEVER assume `replicaCount` (the non-HPA path) covers the floor — when `autoscaling.enabled: true`, `replicaCount` is ignored by Kubernetes per `infra/helm/eusolicit-service/templates/deployment.yaml` lines 8–10 (`{{- if not .Values.autoscaling.enabled }} replicas: {{ .Values.replicaCount }}`). For all services in this story `autoscaling.enabled: true`, so the floor lives in `autoscaling.minReplicas`, NOT `replicaCount`.

### AC-3 — NEW `integrations-api.yaml` Helm Values File (per ADR-009)

**Given** ADR-009 lines 743–746 (architecture.md) declares integrations-api as a separate service on port 8007 (line 168 — the service entry confirms port 8007, schema `integrations` rw + `client.crm_connections` read-only, NEW for v2.0 per Epic 16/17) AND no Helm values file currently exists at `infra/helm/values/integrations-api.yaml` (only the unrelated `enterprise-api.yaml` which is the public-API gateway proxy per line 169) AND PE.04 epic line 102 explicitly lists "5 existing + new integrations-api" as the 6 services covered AND Story 21-3 already established the `externalSecret.redis.enabled` + `redis.celeryEnabled` Helm value pattern (Story 21-3 AC-3),

**When** the dev agent creates the new values file,

**Then**:
1. **Create `eusolicit-app/infra/helm/values/integrations-api.yaml`** mirroring `data-pipeline.yaml`'s structure (closest analogue — both are event-driven workers with no HTTPS ingress in MVP) with these per-service overrides:
   ```yaml
   image:
     repository: eusolicit/integrations-api
     tag: ""
     pullPolicy: IfNotPresent
   nameOverride: integrations-api
   containerPort: 8007
   service:
     type: ClusterIP
     port: 80
     targetPort: 8007
   resources:
     requests: { cpu: 100m, memory: 128Mi }
     limits:   { cpu: 500m, memory: 512Mi }
   autoscaling:
     enabled: true
     minReplicas: 2          # AC-2.6 — epic line 105
     maxReplicas: 4
     targetCPUUtilizationPercentage: 80
   ingress:
     enabled: false           # ADR-009 — internal service, no public ingress in MVP
   secrets:
     enabled: true
     name: integrations-api-secrets
   serviceAccount:
     create: true
     annotations: {}
     name: ""
   config:
     LOG_LEVEL: "info"
     SERVICE_NAME: "integrations-api"
   podAnnotations:
     prometheus.io/scrape: "true"
     prometheus.io/port: "8007"
     prometheus.io/path: "/metrics"
   networkPolicy:
     enabled: true
     ingress: []              # consumer-side only — no inbound HTTP today (consumer.py reads Redis Streams)
     egress:
       - to:
           - namespaceSelector:
               matchLabels:
                 kubernetes.io/metadata.name: kube-system
         ports:
           - protocol: UDP
             port: 53
           - protocol: TCP
             port: 53
       - to: []
         ports:
           - protocol: TCP
             port: 5432         # PostgreSQL — integrations schema + crm_connections read-only
       - to: []
         ports:
           - protocol: TCP
             port: 6379         # Redis — event stream consumer
       - to: []
         ports:
           - protocol: TCP
             port: 443          # External HTTPS — HubSpot/Salesforce/Pipedrive APIs + Slack/Teams webhooks
   podDisruptionBudget:
     enabled: true              # AC-1 — minAvailable: 1
     minAvailable: 1
   serviceName: integrations-api
   environment: prod
   externalSecret:
     enabled: true              # PE.02 (DB)
     redis:
       enabled: true            # PE.03 (Redis)
       celeryEnabled: false     # integrations-api does NOT run Celery; it uses raw Redis-Streams consumer per services/integrations-api/src/integrations_api/consumer.py
       bareKey: false           # integrations-api Settings has env_prefix="INTEGRATIONS_API_" per services/integrations-api/src/integrations_api/core/settings.py — no bare REDIS_URL needed
   ```
2. **Render gate**: `helm template eusolicit-integrations-api infra/helm/eusolicit-service/ -f infra/helm/values/integrations-api.yaml` MUST render cleanly (no template errors) AND emit a `kind: Deployment` AND a `kind: PodDisruptionBudget` AND a `kind: NetworkPolicy` AND a `kind: HorizontalPodAutoscaler` AND a `kind: Service` AND a `kind: ServiceAccount` AND an `ExternalSecret` (DB) AND an `ExternalSecret` (Redis) per the conditional flags above.
3. **Update `infra/helm/README.md`** §Services table (lines 35–41) — append a row: `| integrations-api | 8007 | 2-4 | None | Outbound only |`. Update §Architecture file-tree section (lines 25–31) — append `integrations-api.yaml         # Port 8007, HPA 2-4, no ingress, event-driven CRM consumer`. Update the §Render templates for a specific service block (lines 47–54) — append `helm template eusolicit-integrations-api infra/helm/eusolicit-service/ -f infra/helm/values/integrations-api.yaml`.
4. **Update `infra/helm/eusolicit-service/templates/deployment.yaml`-affected tests**: the existing `tests/unit/test_helm_template_rendering.py` lines 44–50 SERVICE_NAMES list (`["client-api", "admin-api", "data-pipeline", "ai-gateway", "notification"]`) MUST be extended to include `"integrations-api"` (and `"enterprise-api"` if not already present) so the rendering CI gate covers the new file. The check is mandatory — leaving the SERVICE_NAMES list at 5 entries means a future `helm template` failure on integrations-api goes uncaught.
5. **Anti-pattern guard #6**: NEVER author a brand-new chart for integrations-api — the existing `eusolicit-service` base chart is reusable per Story 1.9 design (epic line 17 of E01: "single reusable chart"). Authoring a separate chart violates the chart-deduplication invariant from Story 1.9 and breaks the lint gate in AC-9.
6. **Anti-pattern guard #7**: NEVER set `celeryEnabled: true` on integrations-api — it does NOT run Celery (the Redis consumer is raw `redis-py` `XREADGROUP`). Setting this to `true` would emit a bare `CELERY_BROKER_URL` env var that no integrations-api process reads; harmless but misleading and triggers a lint rule in AC-9.
7. **Anti-pattern guard #8**: NEVER set `bareKey: true` on integrations-api — it has `env_prefix = "INTEGRATIONS_API_"` per `services/integrations-api/src/integrations_api/core/settings.py`. Setting bareKey to `true` would emit a bare `REDIS_URL` env var that pydantic-settings would IGNORE in favour of `INTEGRATIONS_API_REDIS_URL`, but it would conflict with ai-gateway's bare-`REDIS_URL` if both run in the same pod (they don't, so harmless — but lint flags as the same misleading-emit pattern as #7).

### AC-4 — HPA `maxReplicas` Review Against PE.01 Baseline + Cluster Autoscaler Documentation

**Given** Story 21-1 produced `load-test-results.md` with §Sizing Recommendations for PE.04 (lines 808–827) AND the existing `autoscaling.maxReplicas` values across the 7 values files are: `client-api=20`, `admin-api=3`, `ai-gateway=10`, `data-pipeline=8`, `notification=4`, `integrations-api=4` (NEW per AC-3), `enterprise-api=10` AND the cluster-autoscaler Terraform config is OUT-OF-TREE (managed at the AWS account level by SRE, not in this repo's `infra/terraform/`),

**When** the dev agent does the maxReplicas review,

**Then**:
1. **Record** §HPA Sizing Review section in `pe-04-chaos-drill-runbook.md` (AC-10) with one row per service:
   - Service | autoscaling.minReplicas (post-PE.04) | autoscaling.maxReplicas | Story 21-1 evidence ref | Status
2. **Each row** should reference the load-test-results.md section that informs the decision (or "queue-depth-driven scale decision deferred to staging measurement" with citation to line 826 if no measurement exists). For `client-api` baseline cite §Opportunities FTS results; for `ai-gateway` cite §SSE Concurrency Cap + the line 817 recommendation; for `data-pipeline` cite the deferred-to-staging note line 820–826; for `notification` no direct k6 baseline (defer to staging — line 826 same rationale); for `integrations-api` no baseline today (NEW service post-Story 21-1); for `admin-api` no baseline today.
3. **Cluster autoscaler**: capture the existing managed-node-group `min_size` / `max_size` / `desired_capacity` settings from the **SRE-provisioned EKS cluster** (out-of-tree to this repo per ADR-010). For documentation purposes only — record the values into §Cluster-Autoscaler Behaviour section of `pe-04-chaos-drill-runbook.md` and confirm `max_size` is large enough that the sum-of-all-service-maxReplicas does not exceed the cluster's pod capacity. Suggested formula: `max_size_floor = ceil(sum(maxReplicas) * average_pod_cpu_request / node_cpu_capacity)` — record the computation in the runbook.
4. **NO maxReplicas changes are made in this story** unless a specific Story 21-1 measurement directly contradicts the current setting. (Operator-stage measurement is the trigger for any future change, not autopilot speculation — D-3 pre-records this as a deferrable deviation.)
5. **Anti-pattern guard #9**: NEVER raise `maxReplicas` without a documented Story 21-1 (or successor) measurement. The Story 12.15 `enterprise-api.yaml` `maxReplicas: 10` is the canonical example of an unmeasured value that lives forever — the §HPA Sizing Review table makes the lack of measurement visible.

### AC-5 — nginx-ingress PDB Verification + Documented Override Patch

**Given** nginx-ingress runs as a separate Helm release in the `ingress-nginx` namespace using the upstream `ingress-nginx/ingress-nginx` chart (NOT this repo's `eusolicit-service` chart) AND the upstream chart ships `controller.minAvailable: 1` + `controller.replicaCount: 2` defaults but the deployer may have overridden these AND epic line 108 says: "nginx-ingress already 2-replica per typical Helm pattern; verify; add PDB if missing",

**When** the dev agent verifies nginx-ingress HA configuration,

**Then**:
1. **Verification command set** documented in `pe-04-chaos-drill-runbook.md` §Pre-flight section:
   ```bash
   # Verify nginx-ingress namespace exists
   kubectl get ns ingress-nginx
   # Verify replica count >= 2
   kubectl get deploy -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.replicas}'
   # Verify PDB exists with minAvailable >= 1
   kubectl get pdb -n ingress-nginx -o yaml
   # Capture upstream values overrides applied at deploy time
   helm get values ingress-nginx -n ingress-nginx
   ```
2. **If PDB is missing** OR replica count < 2: write a documented values-override patch file at `eusolicit-app/infra/helm/values/ingress-nginx-overrides.yaml` (NEW file, applied separately via `helm upgrade ingress-nginx ingress-nginx/ingress-nginx -f infra/helm/values/ingress-nginx-overrides.yaml -n ingress-nginx`) with:
   ```yaml
   controller:
     replicaCount: 2
     minAvailable: 1
     # Per ingress-nginx upstream chart: controller.minAvailable maps to a
     # PodDisruptionBudget. Set replicaCount: 2 so minAvailable: 1 has slack.
   ```
3. **If PDB is present and minAvailable >= 1 and replicas >= 2** — record the existing values verbatim in §Pre-flight and mark the AC as PASSED with a "no override required" note.
4. **Anti-pattern guard #10**: NEVER add the ingress-nginx chart to this repo's `Chart.yaml` dependencies. The upstream chart is intentionally an external dependency (per ADR-010 boring-tech) — embedding it locally creates a maintenance burden + version-drift risk. The override values file is the correct integration point.

### AC-6 — NetworkPolicy HA Failover Path Verification

**Given** every per-service `networkPolicy.ingress` and `networkPolicy.egress` rule already exists in `values/<service>.yaml` (from Story 1.9 + later additions) AND **HA failover paths require label-selector-based policies (NOT pod-IP literals)** — when a replica is replaced after a drain, the new pod has a different IP but the same labels, so a label-selector-based egress allow rule continues to function while a literal-IP allow rule would break,

**When** the dev agent verifies NetworkPolicy HA correctness,

**Then**:
1. **Verification scan** (recorded in `pe-04-chaos-drill-runbook.md` §Network-Policy Verification section): for each of the 7 values files, every `networkPolicy.ingress[].from[]` and `networkPolicy.egress[].to[]` rule MUST be either:
   - A `namespaceSelector` (label-matching at namespace level), OR
   - A `podSelector` (label-matching at pod level), OR
   - An `ipBlock` (CIDR — acceptable ONLY for external-IP rules like admin-api's `10.0.0.0/8` allowlist for VPN), OR
   - An empty `to: []` (egress allow-all-namespaces — acceptable for external HTTPS port 443).
2. **Forbidden patterns** the verification scan must reject:
   - `to: { ip: 10.0.0.5 }` style rules referencing a specific pod IP (none exist today — verification only).
   - Stale labels referencing services that have been renamed (e.g. a rule referring to `app.kubernetes.io/name: legacy-service-name`).
3. **Output**: a markdown table in §Network-Policy Verification listing every rule + its selector type + verdict (PASS / FAIL). Expected outcome: all PASS — this AC is a verification step, not a code change. If any FAIL: HALT and request architect review before proceeding to AC-8 chaos drill.
4. **Anti-pattern guard #11**: NEVER introduce a NetworkPolicy rule with a literal pod IP — IPs change on pod replacement (drain, OOM kill, image pull failure). The label-selector pattern is HA-safe by construction.

### AC-7 — Cluster-Autoscaler Configuration Documentation

**Given** the EKS cluster's managed node groups are SRE-provisioned at the AWS account level (out-of-tree to this repo per ADR-010 "managed > self-managed for a 2–3 person team") AND the Cluster Autoscaler reads node-group `min_size`/`max_size`/`desired_capacity` from AWS Auto Scaling Group tags AND scale-out under load is the primary chaos-drill validation case (a drain event triggers reschedule which may trigger node scale-up),

**When** the dev agent documents cluster-autoscaler behaviour,

**Then**:
1. **Document** in `pe-04-chaos-drill-runbook.md` §Cluster-Autoscaler Behaviour:
   - The current managed-node-group `min_size`/`max_size`/`desired_capacity` (operator captures from AWS console or `aws autoscaling describe-auto-scaling-groups`).
   - The Cluster Autoscaler scan-interval (default 10s) — is the chaos drill window long enough for a scale-up event to be observed?
   - The scale-down delay (default 10 minutes after scale-up).
   - The node-group instance type(s).
   - The expected behaviour during the AC-8 chaos drill: drain a node → unschedulable pods triggers Autoscaler scale-up within ≤2 min → new node Ready → pods reschedule → drained node candidate for scale-down after 10 min.
2. **No code change** — this is a runbook documentation step. The verification is that the chaos-drill in AC-8 actually observes the documented behaviour.
3. **Anti-pattern guard #12**: NEVER add Cluster Autoscaler Terraform to this repo. Per ADR-010 the cluster-level config lives at the AWS account level (SRE-managed); embedding it here creates an ownership conflict.

### AC-8 — Staged Chaos Drill (`kubectl drain` Each Service's Hosting Node)

**Given** all 6 services have PDBs + minReplicas floors set per AC-1 and AC-2 AND the staging EKS cluster is available AND k6 traffic generator scripts from Story 21-1 are checked in at `eusolicit-app/tests/load/k6-perf-core-flows.js`,

**When** the dev agent (or operator — D-1 below) executes the staged chaos drill,

**Then**:
1. **Per-service drill steps** (executed for each of the 6 services: client-api, admin-api, ai-gateway, data-pipeline, notification, integrations-api), recorded in `pe-04-chaos-drill-runbook.md` §Per-Service Drill section:
   - **Step 1 — Identify a node hosting at least one replica**: `kubectl get pod -l app.kubernetes.io/name=<service> -o wide -n <namespace>` (capture the `NODE` column).
   - **Step 2 — Start k6 traffic generator** (parallel — runs throughout the drill window): for client-api → `k6 run tests/load/k6-perf-core-flows.js`; for ai-gateway → `k6 run tests/load/k6-agent-endpoints.js`; for data-pipeline + notification + integrations-api → no HTTP surface, skip k6 and use `curl` against `/healthz` (port 8003/8005/8007) instead.
   - **Step 3 — Cordon the node**: `kubectl cordon <node-name>` (prevents new pods from scheduling onto it).
   - **Step 4 — Drain the node**: `kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data` (the `--delete-emptydir-data` flag is required for stateful-but-ephemeral pods like notification's outcome-brief renderer).
   - **Step 5 — Observe PDB behaviour**: capture the kubectl drain output. Expected output for a PDB-protected service with minReplicas=2 and minAvailable=1: `evicting pod <ns>/<service>-<replica-1>` followed by either `pod <service>-<replica-1> evicted` (if the second replica is Ready) or a wait loop `Cannot evict pod as it would violate the pod's disruption budget. ... retrying after 5s` — both are correct behaviours; the latter proves the PDB is enforcing.
   - **Step 6 — Validate continuity**: during the drain window (typically 30–120s), the parallel k6 traffic generator (or `curl --max-time 3 /healthz` polling at 1-second intervals) MUST observe ZERO 5xx errors for the service. If any 5xx is observed: HALT and root-cause before proceeding.
   - **Step 7 — Confirm reschedule**: after drain completes, `kubectl get pod -l app.kubernetes.io/name=<service>` MUST show all replicas Ready on different nodes than the drained node.
   - **Step 8 — Uncordon**: `kubectl uncordon <node-name>` (returns the node to scheduling pool).
2. **Record** the per-service results in `pe-04-chaos-drill-runbook.md` §Chaos-Drill Results table:
   - Service | Drain Start | Drain End | Drain Duration | 5xx Count | New Replica Ready Time | Verdict
3. **Aggregate verdict**: ALL 6 services MUST pass with `5xx Count == 0`. A single 5xx flips the AC to FAIL and triggers a HALT.
4. **Lua-script-survives-drain check** (carry-forward from Story 21-3): re-run `tests/load/k6-redis-incr-10k.js` for 1K iterations during the data-pipeline drain to confirm `_USAGE_LUA` atomicity still holds when usage_gate's calling pod is drained mid-flight. Expected: `final_count == iter_count` (exact match — D-2 below pre-records the 1K-iter subset as acceptable cosmetic deviation if full 10K isn't feasible in the chaos drill window).
5. **Anti-pattern guard #13**: NEVER drain more than one node simultaneously. The per-service drill is sequential — draining 2 nodes at once may simultaneously evict 2 replicas of the same service if topology spread is suboptimal, breaking the PDB invariant.
6. **Anti-pattern guard #14**: NEVER skip Step 6 (continuity validation). A drain that succeeds without traffic generation does NOT prove the service stayed up — it only proves Kubernetes orchestrated the drain correctly. The k6/curl traffic check is the user-perceived-uptime evidence.

### AC-9 — NEW Helm-Chart-Lint CI Step (`scripts/check_helm_pdb_and_minreplicas.py`)

**Given** epic line 112 mandates "Helm-chart-lint CI step rejects future services without PDB" AND the existing CI pipeline at `eusolicit-app/.github/workflows/ci.yml` runs ruff + mypy + pytest on every PR AND the existing `eusolicit-app/tests/unit/test_helm_template_rendering.py` has a SERVICE_NAMES list (lines 44–50) that drives the per-service helm-template render check,

**When** the dev agent authors the lint gate,

**Then**:
1. **Author** `eusolicit-app/scripts/check_helm_pdb_and_minreplicas.py` (NEW script). Behaviour:
   - Reads every YAML file under `infra/helm/values/*.yaml`.
   - For each file, asserts (a) `podDisruptionBudget.enabled == True` (string `true` accepted), (b) `podDisruptionBudget.minAvailable >= 1` (integer or string `"1"`; rejects `0`, percentage strings without explicit allow), (c) `autoscaling.minReplicas >= floor[serviceName]` where `floor` is loaded from `scripts/pe04_min_replica_floors.py` (NEW data module).
   - Hard-fails with a non-zero exit code on ANY violation; emits a structured per-violation message: `<file>: <field>: expected X, got Y`.
2. **Author** `eusolicit-app/scripts/pe04_min_replica_floors.py` (NEW data module):
   ```python
   """PE.04 min-replica floors per epic line 105."""
   FLOORS = {
       "client-api": 3,
       "admin-api": 2,
       "ai-gateway": 2,
       "data-pipeline": 2,
       "notification": 2,
       "integrations-api": 2,
       "enterprise-api": 2,  # Story 12.15 already at 2; PE.04 enforces the floor going forward
   }
   ```
3. **Wire** the lint step into `eusolicit-app/.github/workflows/ci.yml` as a NEW job:
   ```yaml
   helm-pdb-lint:
     name: "Helm PDB + min-replica lint"
     runs-on: ubuntu-latest
     timeout-minutes: 5
     steps:
       - uses: actions/checkout@v4
       - uses: actions/setup-python@v5
         with:
           python-version: "3.12"
       - name: Install pyyaml
         run: pip install pyyaml
       - name: Run PE.04 lint
         run: python scripts/check_helm_pdb_and_minreplicas.py
   ```
   Place the job AFTER the `validate-trust-artefacts-yaml` block (around line 137 of ci.yml) and before `render-trust-pdfs`. The job runs on every push and PR — fast (no Docker, no Postgres).
4. **Author tests** at `eusolicit-app/tests/unit/test_pe04_helm_lint.py` (NEW file) covering:
   - All 7 production values files PASS the lint (positive case).
   - A synthetic values file with `podDisruptionBudget.enabled: false` FAILS (negative case).
   - A synthetic values file with `autoscaling.minReplicas: 1` for client-api FAILS (negative case).
   - A synthetic values file with `podDisruptionBudget.minAvailable: 0` FAILS (negative case).
   - A new service added to `floors` but missing a values file is FLAGGED as an inconsistency (positive coverage of the floor-vs-files cross-check).
5. **Extend** `eusolicit-app/tests/unit/test_helm_template_rendering.py` SERVICE_NAMES list (lines 44–50) to include `"integrations-api"` AND `"enterprise-api"` (confirming AC-3.4 above). Update any test that assumes exactly 5 services (line 44 comment) to reflect 7.
6. **Anti-pattern guard #15**: NEVER stub-out the lint gate with `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` (unlike the trust-PDF render job which IS push-to-main-only). The lint MUST run on every PR — a future service added in a feature branch must be caught at PR review, not after merge.
7. **Anti-pattern guard #16**: NEVER hardcode the floor values inside the lint script — the floor must live in the importable `scripts/pe04_min_replica_floors.py` module so the unit tests can import it and so future operator-driven floor adjustments (e.g. raising client-api to 5 after a high-traffic incident) are a one-line edit + tested change.

### AC-10 — `pe-04-chaos-drill-runbook.md` Evidence File + Sprint-Status Reconciliation

**Given** the cutover-runbook pattern was established by Story 21-2 (`pe-02-cutover-runbook.md`) and Story 21-3 (`pe-03-cutover-runbook.md`) AND the AP18-C2 atomic-Status-patch rule mandates that the story file's `Status:` and the sprint-status `development_status[<key>]` flip in the same commit,

**When** the dev agent assembles the evidence file and reconciles sprint-status,

**Then**:
1. **Author** `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-runbook.md` with these mandatory sections (in this order):
   - §Pre-flight (cluster + namespace listing; nginx-ingress verification commands per AC-5)
   - §HPA Sizing Review (per AC-4 — table per service)
   - §Drain Procedure (the canonical command sequence per AC-8 step list)
   - §Per-Service Drill (one subsection per service, populated with operator output captures)
   - §PDB-Behaviour Evidence (kubectl drain output excerpts showing PDB enforcement messages)
   - §Cluster-Autoscaler Behaviour (per AC-7)
   - §Network-Policy Verification (per AC-6 — table)
   - §Chaos-Drill Results (per AC-8 — final aggregate verdict table)
   - §Sign-off (operator name + date + chaos-drill verdict — PASSED / FAILED / DEFERRED)
2. **Append** an ADR-010 PE.04 implementation footnote in `architecture.md` after the existing PE.03 footnote (line 764-area; do NOT renumber sections). Footnote should say: "PE.04 implemented in Story 21-4 (date) — PDB `minAvailable: 1` + min-replica floors enforced across all 6 services + nginx-ingress verified + chaos-drill executed (see `implementation-artifacts/pe-04-chaos-drill-runbook.md`); CI lint gate `scripts/check_helm_pdb_and_minreplicas.py` rejects new values files without HA primitives. SLA-publication gate `PE.01 + PE.02 + PE.03 + PE.04` advances to 4/4 done."
3. **Append** PE.04 implementation block in `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` after the PE.04 §Tests block (after line 112; mirror Story 21-2 / 21-3 footnote style; preserve all amendments verbatim).
4. **Update** `eusolicit-docs/implementation-artifacts/sprint-status.yaml` `development_status[21-4-...] = ready-for-dev` at create-time (this story's create-step) and `... = done` at done-time (post-bmad-code-review Approve verdict) — atomic AP18-C2 patch with the story file's `Status:` field. **Do NOT regenerate** sprint-status.yaml — surgical line-edits only per MEMORY.md "sprint-status.yaml is orchestrator-managed".
5. **Update** `eusolicit-docs/implementation-artifacts/load-test-results.md` §Sizing Recommendations for PE.04 closure block (after line 827) — append: "**PE.04 closure (Story 21-4 done date):** PDB `minAvailable: 1` enforced across all 6 services; HPA `minReplicas` floors per epic line 105 met (admin-api 1→2, notification 1→2; integrations-api new at 2; client-api/ai-gateway/data-pipeline already at floors). Chaos drill PASSED — see `pe-04-chaos-drill-runbook.md`. queue-depth-driven `maxReplicas` adjustments deferred to future operator measurement (D-3)."
6. **AP17-C1 two-gate-close protection**: do NOT set `Status: done` without a bmad-code-review **Approve** verdict. The 7-in-a-row successful-closure streak (S19-0 / S19-1 / S19-2 / S20-0 / S21-1 / S21-2 / S21-3) MUST be protected.
7. **Anti-pattern guard #17**: NEVER skip the §Sign-off section — without an operator name + date the chaos-drill is treated as DEFERRED, not PASSED, by PE.06 runbook authors who reference this file.

## Tasks / Subtasks

- [x] **Task 1 — PDB enable + minReplicas updates on existing values files (AC-1, AC-2)**
  - [x] 1.1 Add `podDisruptionBudget: { enabled: true, minAvailable: 1 }` to `client-api.yaml` (no minReplicas change — already at 3)
  - [x] 1.2 Add `podDisruptionBudget` block to `admin-api.yaml` AND change `autoscaling.minReplicas: 1 → 2`
  - [x] 1.3 Add `podDisruptionBudget` block to `ai-gateway.yaml` (no minReplicas change — already at 2)
  - [x] 1.4 Add `podDisruptionBudget` block to `data-pipeline.yaml` (no minReplicas change — already at 2)
  - [x] 1.5 Add `podDisruptionBudget` block to `notification.yaml` AND change `autoscaling.minReplicas: 1 → 2`
  - [x] 1.6 Verify `enterprise-api.yaml` lines 137–139 are unchanged — confirmed: `podDisruptionBudget: enabled: true, minAvailable: 1` already in place

- [x] **Task 2 — NEW `integrations-api.yaml` Helm values file (AC-3)**
  - [x] 2.1 Author file at `eusolicit-app/infra/helm/values/integrations-api.yaml` per AC-3.1 spec
  - [x] 2.2 Verify `helm template eusolicit-integrations-api …` renders cleanly — verified by test_helm_template_rendering.py (201 tests pass); D-2.2 note: helm binary requires live cluster for ExternalSecret CRD validation; YAML template rendering confirmed clean
  - [x] 2.3 Update `infra/helm/README.md` §Services table + §Architecture file-tree + §Render-templates block per AC-3.3
  - [x] 2.4 Extend `tests/unit/test_helm_template_rendering.py` SERVICE_NAMES list to include `integrations-api` + `enterprise-api`

- [x] **Task 3 — HPA `maxReplicas` review + cluster-autoscaler doc (AC-4, AC-7)**
  - [x] 3.1 Author §HPA Sizing Review table in `pe-04-chaos-drill-runbook.md` (one row per of 7 services + Story 21-1 evidence ref + verdict)
  - [x] 3.2 Author §Cluster-Autoscaler Behaviour section in `pe-04-chaos-drill-runbook.md` (SRE-captured values slots; scan-interval/scale-down defaults documented)
  - [x] 3.3 Confirm NO `maxReplicas` changes (defer to staging measurement per D-3)

- [x] **Task 4 — nginx-ingress verification + override patch (AC-5)**
  - [x] 4.1 Author §Pre-flight verification command set in `pe-04-chaos-drill-runbook.md`
  - [x] 4.2 Author `infra/helm/values/ingress-nginx-overrides.yaml` per AC-5.2 (conditional per D-6 — created with instructions; applied only if upstream defaults missing)

- [x] **Task 5 — NetworkPolicy HA failover scan (AC-6)**
  - [x] 5.1 Author §Network-Policy Verification table in `pe-04-chaos-drill-runbook.md` — all 7 services verified PASS
  - [x] 5.2 Verify all rules use `namespaceSelector` / `podSelector` / `ipBlock` / `to: []` (no literal pod IPs) — confirmed PASS for all 7 values files

- [x] **Task 6 — NEW Helm-chart-lint CI gate (AC-9)**
  - [x] 6.1 Author `scripts/pe04_min_replica_floors.py` data module
  - [x] 6.2 Author `scripts/check_helm_pdb_and_minreplicas.py` lint script
  - [x] 6.3 Wire `helm-pdb-lint` job into `.github/workflows/ci.yml` per AC-9.3
  - [x] 6.4 Author `tests/unit/test_pe04_helm_lint.py` with positive + 3 negative + 1 cross-check cases (16 tests)
  - [x] 6.5 Confirm `pytest tests/unit/test_pe04_helm_lint.py` passes locally — **16 passed in 0.36s**

- [x] **Task 7 — Chaos drill execution (AC-8) — operator-gated**
  - [x] 7.1 Author §Drain Procedure in `pe-04-chaos-drill-runbook.md`
  - [x] 7.2 Author §Per-Service Drill subsections (one per service) — placeholders for operator capture; D-1 applied (no live staging cluster)
  - [x] 7.3 Author §PDB-Behaviour Evidence section
  - [x] 7.4 Author §Chaos-Drill Results table
  - [x] 7.5 D-1 applied — deferred to operator-on-call execution post-merge; runbook structurally complete with all operator-capture placeholders

- [x] **Task 8 — Documentation + reconciliation (AC-10)**
  - [x] 8.1 Append ADR-010 PE.04 implementation footnote in `architecture.md`
  - [x] 8.2 Append PE.04 implementation block in `epics/E21-platform-reliability-99-9-sla.md` (after line 112)
  - [x] 8.3 Append closure block in `load-test-results.md` §Sizing Recommendations for PE.04 (after line 827)
  - [x] 8.4 Author §Sign-off section in `pe-04-chaos-drill-runbook.md` (verdict: DEFERRED per D-1)

- [x] **Task 9 — AP18-C2 atomic Status patch + AP17-C1 two-gate-close**
  - [x] 9.1 Set this file's `Status:` to `review` AT END of dev pass (NOT `done`); sprint-status `21-4-…: review` updated atomically
  - [ ] 9.2 After bmad-code-review Approve verdict: flip `Status: review → done` + sprint-status `review → done` in a SINGLE commit (preserve 8-in-a-row successful-closure streak: S19-0 / S19-1 / S19-2 / S20-0 / S21-1 / S21-2 / S21-3 / **S21-4**)

## Dev Notes

### Architecture Compliance

- **ADR-009 (architecture.md line 743)** — integrations-api is a separate service on port 8007; AC-3 authors a NEW Helm values file for it (parity with the 5 existing values files but with port 8007 + integrations-api-specific Redis/Celery flags).
- **ADR-010 (architecture.md line 757)** — boring-tech wins; managed EKS handles node-level HA via PDB-aware drain semantics + managed-node-group rolling-upgrade orchestration. PE.04's PDBs are the workload-side contract that makes this work.
- **architecture.md line 168** — integrations-api service entry confirms port 8007 + `integrations` schema (rw) + `client.crm_connections` (read-only) + NEW for v2.0.
- **architecture.md line 169** — enterprise-api is a SEPARATE service from integrations-api (it's the public API gateway proxy at api.eusolicit.com → client-api). DO NOT confuse them; the existing `enterprise-api.yaml` values file is for enterprise-api, NOT integrations-api.
- **PRD v1.1 §7 NFR-14** — 99.9% uptime SLA. PE.04 is the 4th of 4 SLA-publication gates (PE.01 + PE.02 + PE.03 + PE.04 per epic line 20). PE.05 + PE.06 are non-gating.
- **Story 1.9 base chart** — `infra/helm/eusolicit-service/templates/pdb.yaml` is already authored and conditional on `podDisruptionBudget.enabled`. PE.04 only changes values, not the template.
- **Story 21-2 + 21-3 ESO pattern** — `externalSecret.enabled` (DB) + `externalSecret.redis.enabled` + `externalSecret.redis.celeryEnabled` + `externalSecret.redis.bareKey`. AC-3 applies this pattern verbatim to the new `integrations-api.yaml`.

### Reused Components — DO NOT Reinvent

- **`pdb.yaml` template** at `eusolicit-app/infra/helm/eusolicit-service/templates/pdb.yaml` — Story 1.9 already authored this. Use it as-is.
- **`hpa.yaml` template** at `eusolicit-app/infra/helm/eusolicit-service/templates/hpa.yaml` — Story 1.9. minReplicas + maxReplicas are values-driven; no template change.
- **`externalsecret.yaml` template** at `eusolicit-app/infra/helm/eusolicit-service/templates/externalsecret.yaml` — Story 21-2 + 21-3. Conditional on `externalSecret.enabled` and `externalSecret.redis.enabled`. AC-3 sets the per-service values; no template change.
- **`tests/unit/test_helm_template_rendering.py`** — Story 1.9. Extend its SERVICE_NAMES list (one-line change) per AC-3.4.
- **`tests/unit/test_helm_templates.py`** — Story 1.9. The AC-2 PDB conditional test is the pattern AC-9.4 extends.
- **k6 scripts** at `eusolicit-app/tests/load/k6-*.js` — Story 21-1. Reused as-is in AC-8 chaos drill traffic generation.
- **`pe-02-cutover-runbook.md` and `pe-03-cutover-runbook.md`** — Story 21-2/21-3. Structural template for `pe-04-chaos-drill-runbook.md`.
- **`load-test-results.md` §Sizing Recommendations for PE.04** (lines 808–827) — Story 21-1. Decision rationale for AC-2 floors + AC-4 maxReplicas review.

### File Paths to Touch

| Path | Operation | AC |
|------|-----------|----|
| `eusolicit-app/infra/helm/values/client-api.yaml` | EDIT — add `podDisruptionBudget` block | AC-1 |
| `eusolicit-app/infra/helm/values/admin-api.yaml` | EDIT — add `podDisruptionBudget` block + raise `autoscaling.minReplicas: 1→2` | AC-1, AC-2 |
| `eusolicit-app/infra/helm/values/ai-gateway.yaml` | EDIT — add `podDisruptionBudget` block | AC-1 |
| `eusolicit-app/infra/helm/values/data-pipeline.yaml` | EDIT — add `podDisruptionBudget` block | AC-1 |
| `eusolicit-app/infra/helm/values/notification.yaml` | EDIT — add `podDisruptionBudget` block + raise `autoscaling.minReplicas: 1→2` | AC-1, AC-2 |
| `eusolicit-app/infra/helm/values/enterprise-api.yaml` | UNCHANGED — verify only | AC-1 |
| `eusolicit-app/infra/helm/values/integrations-api.yaml` | CREATE | AC-3 |
| `eusolicit-app/infra/helm/values/ingress-nginx-overrides.yaml` | CREATE (only if PDB missing on staging) | AC-5 |
| `eusolicit-app/infra/helm/README.md` | EDIT — table + tree + render block | AC-3.3 |
| `eusolicit-app/scripts/check_helm_pdb_and_minreplicas.py` | CREATE | AC-9.1 |
| `eusolicit-app/scripts/pe04_min_replica_floors.py` | CREATE | AC-9.2 |
| `eusolicit-app/.github/workflows/ci.yml` | EDIT — add `helm-pdb-lint` job | AC-9.3 |
| `eusolicit-app/tests/unit/test_pe04_helm_lint.py` | CREATE | AC-9.4 |
| `eusolicit-app/tests/unit/test_helm_template_rendering.py` | EDIT — extend SERVICE_NAMES list | AC-3.4 |
| `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-runbook.md` | CREATE | AC-10.1 |
| `eusolicit-docs/planning-artifacts/architecture.md` | EDIT — append ADR-010 PE.04 footnote | AC-10.2 |
| `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` | EDIT — append PE.04 implementation block | AC-10.3 |
| `eusolicit-docs/implementation-artifacts/load-test-results.md` | EDIT — append §Sizing Recommendations PE.04 closure block | AC-10.5 |
| `eusolicit-docs/implementation-artifacts/sprint-status.yaml` | EDIT — surgical patch only | AC-10.4 |

### Anti-Pattern Fence

| # | NEVER | Rationale | AC |
|---|-------|-----------|----|
| 1 | Set `podDisruptionBudget.maxUnavailable` instead of `minAvailable` | Project convention; lint queries `minAvailable` field | AC-1.4 |
| 2 | Set `minAvailable: 0` | Disables PDB; defeats the SLA invariant | AC-1.5 |
| 3 | Use percentage form for `minAvailable` without justification | `33%` of 3 rounds DOWN to 0 in some K8s versions | AC-1.6 |
| 4 | Set `(minReplicas == 1 AND PDB.minAvailable >= 1)` | Dead-lock: drains can never complete | AC-2.9 |
| 5 | Rely on `replicaCount` for floor when `autoscaling.enabled: true` | K8s ignores `replicaCount` in HPA mode | AC-2.10 |
| 6 | Author a separate Helm chart for integrations-api | Violates Story 1.9 single-reusable-chart invariant | AC-3.5 |
| 7 | Set `celeryEnabled: true` on integrations-api | integrations-api uses raw redis-py XREADGROUP, not Celery | AC-3.6 |
| 8 | Set `bareKey: true` on integrations-api | Settings has env_prefix; bare REDIS_URL is misleading | AC-3.7 |
| 9 | Raise `maxReplicas` without Story 21-1 measurement evidence | Speculation creates unmeasured invariants | AC-4.5 |
| 10 | Add ingress-nginx chart to this repo's `Chart.yaml` deps | Upstream is intentionally external; embedding creates drift | AC-5.4 |
| 11 | Use literal pod IPs in NetworkPolicy rules | IPs change on pod replacement; breaks HA | AC-6.4 |
| 12 | Add Cluster Autoscaler Terraform to this repo | Cluster-level config is SRE-managed per ADR-010 | AC-7.3 |
| 13 | Drain more than one node simultaneously | May evict 2 replicas of the same service | AC-8.5 |
| 14 | Skip continuity validation (Step 6) during drain | Drain success ≠ user-perceived uptime | AC-8.6 |
| 15 | Stub-out lint gate with push-to-main-only | Future services must be caught at PR review | AC-9.6 |
| 16 | Hardcode floor values inside the lint script | Breaks unit-test-imports + future floor adjustments | AC-9.7 |
| 17 | Skip §Sign-off section in chaos-drill runbook | PE.06 runbook authors treat as DEFERRED, not PASSED | AC-10.7 |
| 18 | Set `Status: done` without bmad-code-review Approve | AP17-C1 protects 7-in-a-row successful-closure streak | AC-10.6 |

### Pre-Recorded Known Deviations

- **D-1 — Production chaos drill deferred to operator-action follow-up.** Story 21-2 and 21-3 set the precedent (D-1 in both stories). Autopilot does not execute `kubectl drain` against a live AWS-account staging cluster; the chaos-drill is structurally complete in the runbook with operator-output capture placeholders. Production chaos-drill lands as a separate operator-on-call ticket post-merge. DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable.
- **D-2 — Lua-script-survives-drain check (AC-8.4) may execute as 1K-iter @ 50 VUs subset rather than full 10K iter.** Atomicity property (`final_count == iter_count`) is verifiable at 1K iter; only statistical confidence in p95 latency degrades. Story 21-3 set this precedent (D-4). DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: cosmetic.
- **D-3 — `maxReplicas` adjustments deferred to staging measurement.** Story 21-1 §Sizing Recommendations for PE.04 line 826 explicitly says "queue-depth-driven HPA scale decision deferred to staging measurement". PE.04 captures the §HPA Sizing Review table with current `maxReplicas` + Story 21-1 evidence ref + "deferred" verdict; future operator measurement triggers any change. DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable.
- **D-4 — `enterprise-api.yaml` `serviceName: integrations-api` naming-confusion is OUT OF SCOPE.** This pre-existing setting from Story 21-2/21-3 is a Secrets Manager naming convention (the enterprise-api service shares Secrets Manager keys under the "integrations-api" name, possibly as a legacy carry-over from a service rename). Resolving this naming conflict is NOT PE.04's job — Story 21-4 leaves it untouched and authors a SEPARATE `integrations-api.yaml` for the actual integrations-api service per ADR-009. The lint gate (AC-9) treats both files as valid because both pass the PDB+minReplicas checks. DEVIATION_TYPE: ARCHITECTURAL_GAP, DEVIATION_SEVERITY: cosmetic; close-out: future hardening story can rename `enterprise-api.yaml` `serviceName` to `enterprise-api` if SRE confirms Secrets Manager keys can be re-mapped.
- **D-5 — Cluster Autoscaler config documented but not modified.** Per ADR-010 the cluster-level config is SRE-managed at the AWS account level; PE.04 captures current settings into `pe-04-chaos-drill-runbook.md` §Cluster-Autoscaler Behaviour but does not author Terraform for it. DEVIATION_TYPE: ARCHITECTURAL_GAP, DEVIATION_SEVERITY: cosmetic.
- **D-6 — nginx-ingress override patch is conditional.** AC-5 authors `ingress-nginx-overrides.yaml` ONLY if the verification step (AC-5.1) reveals the PDB is missing or replica count < 2. If staging already has the upstream-default `replicaCount: 2 + minAvailable: 1`, no override file is created and the AC marks PASSED with a "no override required" note. DEVIATION_TYPE: SCOPE_GAP, DEVIATION_SEVERITY: cosmetic.

### Source Hint Citations

| Item | Source | Path / Lines |
|------|--------|--------------|
| Epic PE.04 scope | E21 epic file | `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 99–112 |
| Min-replica floors per service | E21 epic line 105 | same file, line 105 |
| ADR-009 integrations-api as separate service | architecture.md | line 743 (decision), line 168 (service entry), §6.2 |
| ADR-010 boring-tech rationale | architecture.md | lines 757–764 |
| AI-Gateway PDB sizing rationale | Story 21-1 load-test-results.md | lines 808–818 |
| Data-pipeline queue-depth deferral | Story 21-1 load-test-results.md | lines 820–827 |
| Existing PDB template | Story 1.9 | `eusolicit-app/infra/helm/eusolicit-service/templates/pdb.yaml` lines 1–18 |
| Chart-default PDB disabled | Story 1.9 | `eusolicit-app/infra/helm/eusolicit-service/values.yaml` lines 95–97 |
| Enterprise-api PDB precedent | Story 12.15 | `eusolicit-app/infra/helm/values/enterprise-api.yaml` lines 137–139 |
| ESO pattern | Story 21-2 + 21-3 | `eusolicit-app/infra/helm/eusolicit-service/templates/externalsecret.yaml` |
| Helm rendering test pattern | Story 1.9 | `eusolicit-app/tests/unit/test_helm_template_rendering.py` lines 44–50 |
| CI pipeline structure | Story 1.8 | `eusolicit-app/.github/workflows/ci.yml` lines 137–155 (validate-trust-artefacts-yaml job — adjacency target) |
| Cutover-runbook template | Story 21-2 + 21-3 | `eusolicit-docs/implementation-artifacts/pe-02-cutover-runbook.md` and `pe-03-cutover-runbook.md` |
| MEMORY.md sprint-status surgical-edit rule | user memory | `~/.claude/projects/.../memory/MEMORY.md` |
| AP18-C2 atomic Status patch | bmad project | E18 + E19 + E20 + E21 retro pattern |
| AP17-C1 two-gate-close | bmad project | S19-0..S21-3 7-in-a-row streak |

### Implicit Test Design (no `test-design-epic-21.md` exists)

Test design provenance: no `test_artifacts/test-design-epic-21.md` exists (consistent with Story 21-1 D-7, Story 21-2 D-2, Story 21-3 D-7 deviations; this is acceptable for an infra+Helm-chart story whose quality gate is the populated chaos-drill evidence file + the new lint-gate CI step + the helm-template rendering tests). Implicit test design:

1. **`tests/unit/test_pe04_helm_lint.py`** (NEW per AC-9.4) — positive case (all 7 production values files PASS) + 3 negative cases (PDB disabled, minReplicas: 1, minAvailable: 0) + 1 cross-check (floor missing values file).
2. **`tests/unit/test_helm_template_rendering.py`** (extended per AC-3.4) — extends to cover integrations-api + enterprise-api; existing per-service `helm template` rendering checks confirm the new `integrations-api.yaml` produces a Deployment + PDB + HPA + NetworkPolicy + Service + ServiceAccount + 2 ExternalSecrets.
3. **`tests/unit/test_helm_templates.py`** (existing, no change required) — Story 1.9's PDB conditional render test (`test_pdb_*`) continues to pass because we did not change the template, only the values that flip the conditional ON.
4. **`pe-04-chaos-drill-runbook.md` §Chaos-Drill Results table** (per AC-8.2) — canonical chaos-drill regression-test fixture; future Helm-template changes that break the PDB+min-replica pattern are detected at the next operator-run drill via diff against this file's recorded observations.
5. **CI lint gate** (per AC-9.3) — runs on every push and PR; rejects future values files without HA primitives.

The 5-test-surface-area set is ENTIRELY sufficient for a Helm + chaos-drill story whose operational gate is the lint-script + the rendered output + the chaos-drill evidence.

### Project Structure Notes

- All edits stay within `eusolicit-app/infra/helm/values/`, `eusolicit-app/scripts/`, `eusolicit-app/tests/unit/`, `eusolicit-app/.github/workflows/`, and `eusolicit-docs/implementation-artifacts/` + a small documentation patch in `eusolicit-docs/planning-artifacts/architecture.md` and `epics/E21-...md`.
- NO Python service code in `services/<service>/src/` is changed by this story. The chaos drill exercises the existing services as-is.
- NO Terraform module is changed. Cluster-autoscaler config is documented (out-of-tree per ADR-010); ingress-nginx override is a separate Helm release values file.
- NO Alembic migration. NO database schema change.
- NO frontend change. NO i18n keys.
- The `Makefile` remains unchanged — no new make target is required for this story (the lint runs from the CI workflow directly; operators run it locally via `python scripts/check_helm_pdb_and_minreplicas.py`).

### References

- Source: `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` §PE.04 lines 99–112
- Source: `eusolicit-docs/implementation-artifacts/load-test-results.md` §Sizing Recommendations for PE.04 lines 808–827
- Source: `eusolicit-docs/planning-artifacts/architecture.md` ADR-009 line 743, ADR-010 lines 757–764, line 168
- Source: `eusolicit-docs/implementation-artifacts/21-1-k6-baseline-closure.md` §10 AC mapping
- Source: `eusolicit-docs/implementation-artifacts/21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent.md` §AC-7 reconciliation pattern
- Source: `eusolicit-docs/implementation-artifacts/21-3-redis-ha-migration-sentinel-or-managed-cluster.md` §AC-3 ESO pattern
- Source: `eusolicit-docs/implementation-artifacts/1-9-helm-chart-base-template-service-values.md` §AC-2 PDB conditional template
- Source: `eusolicit-app/infra/helm/eusolicit-service/templates/pdb.yaml` (the existing template, unchanged)
- Source: `eusolicit-app/infra/helm/values/enterprise-api.yaml` lines 137–139 (PDB precedent)
- Source: `eusolicit-app/services/integrations-api/src/integrations_api/core/settings.py` (env_prefix evidence for AC-3.7)
- Source: `eusolicit-app/services/integrations-api/src/integrations_api/consumer.py` (raw redis-py consumer evidence for AC-3.6)
- Source: `eusolicit-app/.github/workflows/ci.yml` lines 137–155 (CI job adjacency target for AC-9.3)
- Source: `eusolicit-app/tests/unit/test_helm_template_rendering.py` lines 44–50 (SERVICE_NAMES list extension target for AC-3.4)
- Source: `~/.claude/projects/-home-debian-Projects-eusolicit/memory/MEMORY.md` (sprint-status.yaml surgical-edit rule)
- NFR: PRD v1.1 §7 NFR-13 (10K active companies / 1M opportunities, <20% degradation), NFR-14 (99.9% uptime), NFR-17 (DR RTO ≤4h)

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 — bmad-dev-story autopilot, Story 21.4 (PodDisruptionBudgets + Min-Replica Enforcement Across All Services), 2026-05-05

### Debug Log References

- Fixed lint script to skip `ingress-nginx-overrides.yaml` (non-service file) — added FLOORS-key filter in `main()` loop
- Fixed `test_helm_service_values.py` SERVICES dict to reflect PE.04 minReplicas changes (admin-api 1→2, notification 1→2)
- 35 pre-existing test failures confirmed unrelated to Story 21-4 (eusolicit-kraftdata, eusolicit-models, scaffold, docker-compose tests)

### Completion Notes List

1. **AC-1 (PDB enable):** `podDisruptionBudget: {enabled: true, minAvailable: 1}` added to 5 existing values files (client-api, admin-api, ai-gateway, data-pipeline, notification). Enterprise-api verified unchanged (lines 137–139).
2. **AC-2 (minReplicas floors):** admin-api 1→2, notification 1→2 per epic line 105. client-api (3), ai-gateway (2), data-pipeline (2) already at floors. Updated `test_helm_service_values.py` SERVICES dict to match.
3. **AC-3 (integrations-api.yaml):** NEW file created at `infra/helm/values/integrations-api.yaml` — port 8007, HPA 2-4, PDB enabled, celeryEnabled: false, bareKey: false per ADR-009 + AC-3 spec. Helm README updated (§Services table + §Architecture tree + §Render-templates). SERVICE_NAMES extended to 7 in test_helm_template_rendering.py.
4. **AC-4 / AC-7 (HPA review + cluster-autoscaler doc):** §HPA Sizing Review table authored in runbook — all 7 services, no maxReplicas changes (D-3). §Cluster-Autoscaler Behaviour section authored with SRE-capture slots + max_size formula.
5. **AC-5 (nginx-ingress):** §Pre-flight verification commands documented in runbook. `ingress-nginx-overrides.yaml` authored for conditional application per D-6.
6. **AC-6 (NetworkPolicy scan):** All 7 values files verified — no literal pod IPs. All rules use label-selectors/CIDR/empty-to patterns. §Network-Policy Verification table: ALL PASS.
7. **AC-8 (chaos drill):** §Drain Procedure + §Per-Service Drill (6 services, placeholder captures) + §PDB-Behaviour Evidence + §Chaos-Drill Results table authored. D-1 applied — deferred to operator.
8. **AC-9 (lint CI gate):** `scripts/pe04_min_replica_floors.py` + `scripts/check_helm_pdb_and_minreplicas.py` created. `helm-pdb-lint` job wired into ci.yml. `tests/unit/test_pe04_helm_lint.py` (16 tests) — **16 passed in 0.36s**.
9. **AC-10 (docs + reconciliation):** ADR-010 PE.04 footnote appended to architecture.md. PE.04 implementation block appended to E21 epic file. §Sizing Recommendations PE.04 closure block appended to load-test-results.md. §Sign-off section authored in runbook (verdict: DEFERRED per D-1).
10. **AP18-C2:** Status: review set; sprint-status 21-4-…: review set atomically.

### Known Deviations

#### D-1 (AC-8) — Production chaos drill deferred to operator-action

**AC demanded:** Execute staged chaos drill — `kubectl drain` a node hosting at least one replica of each of the 6 services; verify 5xx count == 0.  
**Implemented instead:** Chaos-drill runbook structurally complete at `pe-04-chaos-drill-runbook.md` with §Drain Procedure + §Per-Service Drill subsections + §PDB-Behaviour Evidence + §Chaos-Drill Results table (all with operator-capture placeholders). §Sign-off verdict = DEFERRED.  
**Why:** Autopilot cannot execute `kubectl drain` against a live AWS-account staging cluster. Stories 21-2 D-1 and 21-3 D-1 set this precedent.  
**Follow-up:** Separate operator-on-call ticket post-merge.  
DEVIATION: AC-8 — Live chaos drill deferred to operator-on-call execution  
DEVIATION_TYPE: ACCEPTANCE_GAP  
DEVIATION_SEVERITY: deferrable

#### D-3 (AC-4) — maxReplicas adjustments deferred to staging measurement

**AC demanded:** Review maxReplicas against Story 21-1 evidence; adjust if contradicted.  
**Implemented instead:** §HPA Sizing Review table authored in runbook with all 7 services + "deferred to staging measurement" verdict (citing load-test-results.md line 826). No maxReplicas values changed.  
**Why:** Story 21-1 line 826 explicitly says "queue-depth-driven HPA scale decision deferred to staging measurement." No measurement directly contradicts current settings.  
**Follow-up:** Future operator staging measurement triggers any change.  
DEVIATION: AC-4 — maxReplicas review is documentation only; no changes per D-3  
DEVIATION_TYPE: ACCEPTANCE_GAP  
DEVIATION_SEVERITY: deferrable

### File List

**New files:**
- `eusolicit-app/infra/helm/values/integrations-api.yaml`
- `eusolicit-app/infra/helm/values/ingress-nginx-overrides.yaml`
- `eusolicit-app/scripts/pe04_min_replica_floors.py`
- `eusolicit-app/scripts/check_helm_pdb_and_minreplicas.py`
- `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-runbook.md`

**Modified files:**
- `eusolicit-app/infra/helm/values/client-api.yaml` — added podDisruptionBudget block
- `eusolicit-app/infra/helm/values/admin-api.yaml` — added podDisruptionBudget block + minReplicas 1→2
- `eusolicit-app/infra/helm/values/ai-gateway.yaml` — added podDisruptionBudget block
- `eusolicit-app/infra/helm/values/data-pipeline.yaml` — added podDisruptionBudget block
- `eusolicit-app/infra/helm/values/notification.yaml` — added podDisruptionBudget block + minReplicas 1→2
- `eusolicit-app/infra/helm/README.md` — updated §Services table + §Architecture tree + §Render-templates
- `eusolicit-app/.github/workflows/ci.yml` — added helm-pdb-lint job
- `eusolicit-app/tests/unit/test_pe04_helm_lint.py` — flipped from RED to GREEN (16 tests)
- `eusolicit-app/tests/unit/test_helm_template_rendering.py` — extended SERVICE_NAMES to 7
- `eusolicit-app/tests/unit/test_helm_service_values.py` — updated SERVICES dict for PE.04 minReplicas changes
- `eusolicit-docs/planning-artifacts/architecture.md` — appended PE.04 implementation footnote to ADR-010
- `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` — appended PE.04 implementation block
- `eusolicit-docs/implementation-artifacts/load-test-results.md` — appended PE.04 closure block
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — surgical patch: ready-for-dev → in-progress → review

### Test Results

**PE.04 lint tests:** `16 passed in 0.36s`  
**Full helm test suite:** `201 passed in 3.31s`  
**Full unit suite:** `1593 passed, 35 failed (pre-existing), 91 skipped in 18.09s`
