# Story 1.10: Terraform Scaffold & Placeholder Modules

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **DevOps engineer managing EU Solicit cloud infrastructure**,
I want **a Terraform project structure with placeholder modules for all cloud resources the platform requires, organized by environment with remote state configuration**,
so that **when actual cloud provisioning begins (post-MVP), the module interfaces, variable contracts, and environment separation are already established, ensuring consistent IaC from day one**.

## Acceptance Criteria

1. **Root module files**: Root at `eusolicit-app/infra/terraform/` contains `main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `backend.tf`
2. **Module directories**: Each module directory contains placeholder `main.tf`, `variables.tf`, `outputs.tf`: `modules/networking`, `modules/kubernetes`, `modules/database`, `modules/redis`, `modules/storage`, `modules/monitoring`
3. **Environment directories**: `environments/dev/`, `environments/staging/`, `environments/prod/` each with `terraform.tfvars` and `main.tf` that references modules
4. **AWS provider**: `providers.tf` configures AWS provider with version constraints (hashicorp/aws ~> 5.0) and default region `eu-central-1`
5. **Remote state**: `backend.tf` configures S3 backend for remote state with placeholder bucket name
6. **Validation passes**: `terraform validate` passes on the root module (no syntax errors)
7. **README**: A comprehensive README in `eusolicit-app/infra/terraform/` explains the module structure, intended usage, and per-environment deployment commands

## Tasks / Subtasks

- [x] Task 1: Create root module files (AC: 1, 4, 5)
  - [x] 1.1 Create `eusolicit-app/infra/terraform/main.tf` — root module that calls all child modules with variable passthrough
  - [x] 1.2 Create `eusolicit-app/infra/terraform/variables.tf` — top-level input variables: `environment`, `aws_region` (default `eu-central-1`), `project_name` (default `eusolicit`), plus per-module variable groups
  - [x] 1.3 Create `eusolicit-app/infra/terraform/outputs.tf` — aggregate outputs from child modules (VPC ID, EKS cluster endpoint, RDS endpoint, ElastiCache endpoint, S3 bucket ARN, monitoring endpoints)
  - [x] 1.4 Create `eusolicit-app/infra/terraform/providers.tf` — AWS provider with `region = var.aws_region`, version constraint `~> 5.0`, required Terraform version `>= 1.5`
  - [x] 1.5 Create `eusolicit-app/infra/terraform/backend.tf` — S3 backend with placeholder bucket `eusolicit-terraform-state`, key `terraform.tfstate`, region `eu-central-1`, DynamoDB lock table `eusolicit-terraform-locks`
  - [x] 1.6 Create `eusolicit-app/infra/terraform/versions.tf` — `terraform { required_version }` block and `required_providers` block (AWS, Kubernetes, Helm, Random providers)

- [x] Task 2: Create networking module (AC: 2)
  - [x] 2.1 Create `modules/networking/main.tf` — TODO placeholders for VPC, public/private subnets, NAT gateway, internet gateway, route tables, security groups
  - [x] 2.2 Create `modules/networking/variables.tf` — inputs: `vpc_cidr`, `availability_zones`, `private_subnet_cidrs`, `public_subnet_cidrs`, `environment`, `project_name`
  - [x] 2.3 Create `modules/networking/outputs.tf` — outputs: `vpc_id`, `private_subnet_ids`, `public_subnet_ids`, `nat_gateway_id`

- [x] Task 3: Create kubernetes module (AC: 2)
  - [x] 3.1 Create `modules/kubernetes/main.tf` — TODO placeholders for EKS cluster, node groups, OIDC provider, cluster addons (CoreDNS, kube-proxy, VPC-CNI), IAM roles
  - [x] 3.2 Create `modules/kubernetes/variables.tf` — inputs: `cluster_name`, `cluster_version`, `vpc_id`, `subnet_ids`, `node_instance_types`, `node_min_size`, `node_max_size`, `node_desired_size`, `environment`
  - [x] 3.3 Create `modules/kubernetes/outputs.tf` — outputs: `cluster_endpoint`, `cluster_name`, `cluster_certificate_authority`, `oidc_provider_arn`

- [x] Task 4: Create database module (AC: 2)
  - [x] 4.1 Create `modules/database/main.tf` — TODO placeholders for RDS PostgreSQL 16 instance, subnet group, parameter group, security group, automated backups
  - [x] 4.2 Create `modules/database/variables.tf` — inputs: `instance_class`, `allocated_storage`, `db_name`, `engine_version` (default `16`), `vpc_id`, `subnet_ids`, `multi_az`, `backup_retention_period`, `environment`
  - [x] 4.3 Create `modules/database/outputs.tf` — outputs: `db_endpoint`, `db_port`, `db_name`, `db_instance_id`

- [x] Task 5: Create redis module (AC: 2)
  - [x] 5.1 Create `modules/redis/main.tf` — TODO placeholders for ElastiCache Redis 7 cluster, subnet group, parameter group, security group
  - [x] 5.2 Create `modules/redis/variables.tf` — inputs: `node_type`, `engine_version` (default `7.0`), `num_cache_nodes`, `vpc_id`, `subnet_ids`, `environment`
  - [x] 5.3 Create `modules/redis/outputs.tf` — outputs: `redis_endpoint`, `redis_port`, `redis_cluster_id`

- [x] Task 6: Create storage module (AC: 2)
  - [x] 6.1 Create `modules/storage/main.tf` — TODO placeholders for S3 buckets (documents, backups, terraform state), bucket policies, versioning, lifecycle rules, encryption
  - [x] 6.2 Create `modules/storage/variables.tf` — inputs: `bucket_prefix`, `enable_versioning`, `lifecycle_rules`, `environment`
  - [x] 6.3 Create `modules/storage/outputs.tf` — outputs: `documents_bucket_arn`, `documents_bucket_name`, `backups_bucket_arn`

- [x] Task 7: Create monitoring module (AC: 2)
  - [x] 7.1 Create `modules/monitoring/main.tf` — TODO placeholders for CloudWatch log groups, alarms, SNS topics, Prometheus workspace (AMP), Grafana workspace (AMG)
  - [x] 7.2 Create `modules/monitoring/variables.tf` — inputs: `log_retention_days`, `alarm_email`, `enable_prometheus`, `enable_grafana`, `environment`
  - [x] 7.3 Create `modules/monitoring/outputs.tf` — outputs: `log_group_arns`, `sns_topic_arn`, `prometheus_endpoint`, `grafana_endpoint`

- [x] Task 8: Create environment directories (AC: 3)
  - [x] 8.1 Create `environments/dev/main.tf` — references all modules with dev-appropriate defaults (smaller instances, single-AZ, minimal replicas)
  - [x] 8.2 Create `environments/dev/terraform.tfvars` — dev values: `environment = "dev"`, `aws_region = "eu-central-1"`, small instance sizes
  - [x] 8.3 Create `environments/staging/main.tf` — references all modules with staging-appropriate defaults (medium instances, multi-AZ)
  - [x] 8.4 Create `environments/staging/terraform.tfvars` — staging values matching architecture's `stage.eusolicit.com` environment
  - [x] 8.5 Create `environments/prod/main.tf` — references all modules with production-appropriate defaults (HA, multi-AZ, larger instances)
  - [x] 8.6 Create `environments/prod/terraform.tfvars` — production values for HA cluster at `eusolicit.com`
  - [x] 8.7 Each environment `main.tf` should include its own `backend.tf` partial config (unique state key per environment)

- [x] Task 9: Validate and document (AC: 6, 7)
  - [x] 9.1 Run `terraform init` (with `-backend=false` to skip remote state) on root module
  - [x] 9.2 Run `terraform validate` on root module — must pass cleanly
  - [x] 9.3 Run `terraform fmt -check` — all files must be properly formatted
  - [x] 9.4 Replace existing `eusolicit-app/infra/terraform/README.md` with comprehensive documentation covering module structure, usage, environment deployment commands, and state management

## Dev Notes

### CRITICAL: Existing Infrastructure -- Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `docker-compose.yml` | Full Docker Compose stack (9+ containers) | **DO NOT TOUCH** |
| `.env.example` | All env vars for services, DB, Redis, MinIO, ClamAV, auth | **DO NOT TOUCH** |
| `.github/workflows/` | CI workflows (ci.yml, test.yml, quality-gates.yml) | **DO NOT TOUCH** |
| `Makefile` | 16+ targets | **DO NOT TOUCH** |
| `services/*/` | All 5 services with FastAPI skeletons | **DO NOT TOUCH** |
| `packages/*/` | All 4 shared packages | **DO NOT TOUCH** |
| `infra/helm/` | Complete Helm chart (Story 1-9) | **DO NOT TOUCH** |
| `infra/terraform/README.md` | Placeholder README | **REPLACE** with comprehensive docs |

### CRITICAL: Path Convention

All files are under `eusolicit-app/infra/terraform/`. The monorepo root is `/home/debian/Projects/eusolicit/`, and the application lives in `eusolicit-app/`. All task paths are relative to `eusolicit-app/`.

### Architecture-Mandated Infrastructure Specifications

These values are authoritative from the solution architecture and MUST be reflected in module variables/defaults:

**AWS Region & Data Residency (ADR 1.9 — GDPR)**:
- Primary region: `eu-central-1` (Frankfurt) — all data MUST reside within EU
- This applies to PostgreSQL, Redis, S3, and all managed services
- No cross-region replication to non-EU regions

**Kubernetes Cluster (Architecture Section 14.1)**:
- EKS cluster hosting 9+ deployments (5 services + frontend + clamav + celery workers)
- Node groups sized for: 3-20 client-api pods, 2-10 ai-gateway pods, 2-8 data-pipeline pods, 1-4 notification pods, 1-3 admin-api pods
- nginx-ingress controller, External Secrets Operator, cert-manager expected as cluster addons

**Database (Architecture Section 4.3, ADR 1.5)**:
- PostgreSQL 16, single instance with 6 schemas
- Multi-AZ for production, single-AZ for dev/staging
- Automated daily backups with pg_dump to S3 (CronJob in K8s)
- Engine: `16.x` (latest minor version)

**Redis (Architecture Section 4.3)**:
- Redis 7, used for Streams (event bus), Celery broker, rate limits, usage counters
- Single node for dev, clustered for production
- Engine: `7.0` (latest minor version)

**Storage (Architecture Section 4.3)**:
- S3 for tender documents, proposals, exports, db-backups
- Bucket naming convention: `eusolicit-<purpose>-<environment>`
- Versioning enabled for document buckets
- Server-side encryption (AES-256 or KMS)
- Document retention: opportunity lifetime + 2 years (lifecycle rules)

**Monitoring (Architecture Section 15)**:
- Prometheus + Grafana for metrics and dashboards
- Loki for centralized log aggregation
- OpenTelemetry + Jaeger for distributed tracing
- CloudWatch as baseline AWS-native monitoring

**Secrets Management (Architecture Section 14.1)**:
- External Secrets Operator syncing from AWS Secrets Manager
- Per-service secret sets: `client-api-secrets`, `admin-api-secrets`, `ai-gateway-secrets`, `data-pipeline-secrets`, `notification-secrets`

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#ADR 1.9, #Section 4.3, #Section 4.4, #Section 14, #Section 15]

### Environment Strategy (Architecture Section 14.3)

| Environment | Purpose | Infrastructure Profile |
|-------------|---------|----------------------|
| `dev` | Developer/CI testing | Small EKS (2 nodes), db.t3.micro, cache.t3.micro, single-AZ |
| `staging` | Integration testing, QA (`stage.eusolicit.com`) | Medium EKS (3 nodes), db.t3.small, cache.t3.small, multi-AZ |
| `prod` | Live SaaS at `eusolicit.com` | HA EKS (5+ nodes), db.r6g.large, cache.r6g.large, multi-AZ, enhanced monitoring |

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 14.3]

### Terraform Design Decisions

1. **Scaffold only**: No actual resources provisioned. All `resource` blocks should be commented out or empty with `# TODO:` comments describing what will be created. The goal is to establish directory structure and module interface contracts.
2. **Terraform >= 1.5**: Use modern HCL syntax. Required providers in a `versions.tf` or inside `providers.tf`.
3. **AWS-first**: Architecture specifies AWS (eu-central-1). Use `hashicorp/aws` provider. Also include `hashicorp/kubernetes` and `hashicorp/helm` providers for EKS addon management.
4. **S3 remote state**: Backend uses S3 with DynamoDB locking. Placeholder bucket/table names — actual bootstrapping is manual.
5. **Module interface contracts**: Each module's `variables.tf` and `outputs.tf` define the contract. These are the important part — they document what each module needs and provides.
6. **`terraform validate` must pass**: Even with placeholder/empty resources, the HCL must be syntactically valid. Use `# TODO` comments inside resource blocks or use empty resource blocks that reference variables to keep validation passing.
7. **No hardcoded values**: All configurable values go through variables with sensible defaults. Environment-specific overrides go in `terraform.tfvars`.

### Terraform File Patterns

Use these conventions consistently:

```hcl
# variables.tf pattern
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name used for resource naming"
  type        = string
  default     = "eusolicit"
}
```

```hcl
# Module call pattern in main.tf
module "networking" {
  source = "./modules/networking"

  environment = var.environment
  project_name = var.project_name
  # ... module-specific variables
}
```

```hcl
# outputs.tf pattern — aggregates child module outputs
output "vpc_id" {
  description = "VPC ID from networking module"
  value       = module.networking.vpc_id
}
```

### File Structure

All Terraform files under `eusolicit-app/infra/terraform/`:

```
eusolicit-app/infra/terraform/
  README.md                          # REPLACE existing placeholder
  main.tf                            # NEW — root module calling child modules
  variables.tf                       # NEW — top-level input variables
  outputs.tf                         # NEW — aggregated outputs
  providers.tf                       # NEW — AWS + K8s + Helm providers
  backend.tf                         # NEW — S3 remote state config
  versions.tf                        # NEW — required terraform/provider versions
  modules/
    networking/
      main.tf                        # NEW — VPC, subnets, NAT, IGW placeholders
      variables.tf                   # NEW — networking inputs
      outputs.tf                     # NEW — networking outputs
    kubernetes/
      main.tf                        # NEW — EKS, node groups, addons placeholders
      variables.tf                   # NEW — kubernetes inputs
      outputs.tf                     # NEW — kubernetes outputs
    database/
      main.tf                        # NEW — RDS PostgreSQL placeholders
      variables.tf                   # NEW — database inputs
      outputs.tf                     # NEW — database outputs
    redis/
      main.tf                        # NEW — ElastiCache Redis placeholders
      variables.tf                   # NEW — redis inputs
      outputs.tf                     # NEW — redis outputs
    storage/
      main.tf                        # NEW — S3 buckets placeholders
      variables.tf                   # NEW — storage inputs
      outputs.tf                     # NEW — storage outputs
    monitoring/
      main.tf                        # NEW — CloudWatch, AMP, AMG placeholders
      variables.tf                   # NEW — monitoring inputs
      outputs.tf                     # NEW — monitoring outputs
  environments/
    dev/
      main.tf                        # NEW — dev env module calls
      terraform.tfvars               # NEW — dev values
    staging/
      main.tf                        # NEW — staging env module calls
      terraform.tfvars               # NEW — staging values
    prod/
      main.tf                        # NEW — prod env module calls
      terraform.tfvars               # NEW — prod values
```

### Testing Expectations

From the epic-level test design (test-design-epic-01.md):

- **P3 test**: `terraform validate` passes on root module — confirms scaffold syntax is valid
- **Risk E01-R-011 (Low, Score 1)**: Terraform scaffold diverges from actual infrastructure needs. Mitigation: monitor and update when modules are implemented. No active test coverage required now.
- `terraform validate` must be runnable in CI without cloud credentials (use `-backend=false` flag during init)
- `terraform fmt -check` should also pass to enforce consistent formatting

[Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P3, #Risk E01-R-011]

### Previous Story Intelligence (Story 1-9)

Story 1-9 (Helm Chart) established these patterns relevant to this story:
- All infra lives under `eusolicit-app/infra/` — Terraform is at `eusolicit-app/infra/terraform/`
- Existing README.md at `eusolicit-app/infra/terraform/README.md` is a placeholder that should be REPLACED
- Helm values files use architecture-mandated service specs (ports, HPA ranges) — Terraform modules should align with the same infrastructure topology
- Story 1-9 dev notes explicitly marked `infra/terraform/` as "DO NOT TOUCH — that is Story 1-10"

### Project Structure Notes

- All paths relative to `eusolicit-app/` unless stated otherwise
- The `infra/terraform/` directory currently contains only a placeholder README.md
- `infra/helm/` (Story 1-9) is a sibling directory — do not modify it
- `infra/postgres/` contains PostgreSQL init scripts (Story 1-3) — do not modify it

### References

- [Source: eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md#S01.10]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#ADR 1.9 Data Residency & GDPR]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 4.3 Data Layer]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 4.4 Infrastructure & DevOps]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 14.1 Kubernetes Topology]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 14.3 Environment Strategy]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section 15 Observability]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P3 Terraform validate]
- [Source: eusolicit-docs/implementation-artifacts/1-9-helm-chart-base-template-service-values.md#Dev Notes]

## Dev Agent Record

### Agent Model Used

Claude Opus 4 (via BMad dev-story workflow)

### Debug Log References

- All 142 ATDD tests pass (135 previously failing → now passing, 4 pre-existing pass, 3 CLI tests now pass with terraform installed)
- terraform init -backend=false: SUCCESS (Terraform v1.9.8)
- terraform validate: SUCCESS ("The configuration is valid")
- terraform fmt -check -recursive: SUCCESS (exit code 0)
- Regression suite: 1170 passed, 19 failed (all 19 are pre-existing ModuleNotFoundError for eusolicit_models/eusolicit_kraftdata packages — completely unrelated to this story)

### Completion Notes List

- Created complete Terraform scaffold at `eusolicit-app/infra/terraform/` with 6 root module files, 6 child modules (18 files), 3 environment configs (6 files), and comprehensive README
- All module variable/output contracts align with architecture-mandated specifications (ADR 1.9 GDPR data residency, Section 4.3 data layer, Section 14.1 K8s topology, Section 14.3 environment strategy, Section 15 observability)
- Resource blocks are commented-out TODO placeholders — no actual cloud resources provisioned
- Module outputs use placeholder values (empty strings, empty lists, port defaults) enabling terraform validate to pass
- Environment sizing follows Architecture Section 14.3: dev=db.t3.micro/cache.t3.micro/single-AZ, staging=db.t3.small/cache.t3.small/multi-AZ, prod=db.r6g.large/cache.r6g.large/multi-AZ/HA
- Each environment main.tf includes its own S3 backend config with unique state key (environments/dev/, environments/staging/, environments/prod/)
- Added terraform artifact entries to .gitignore (.terraform/, *.tfstate, etc.)
- Installed Terraform v1.9.8 to user-local bin for CLI validation
- .terraform.lock.hcl generated and committed for reproducible provider versions

### File List

New files:
- `eusolicit-app/infra/terraform/main.tf` — root module calling all 6 child modules
- `eusolicit-app/infra/terraform/variables.tf` — top-level input variables with defaults
- `eusolicit-app/infra/terraform/outputs.tf` — aggregated outputs from child modules
- `eusolicit-app/infra/terraform/providers.tf` — AWS provider with region = var.aws_region
- `eusolicit-app/infra/terraform/backend.tf` — S3 remote state with DynamoDB locking
- `eusolicit-app/infra/terraform/versions.tf` — required terraform >= 1.5, AWS ~> 5.0, kubernetes ~> 2.23, helm ~> 2.11, random ~> 3.5
- `eusolicit-app/infra/terraform/.terraform.lock.hcl` — provider lock file
- `eusolicit-app/infra/terraform/modules/networking/main.tf` — VPC/subnet/NAT placeholders
- `eusolicit-app/infra/terraform/modules/networking/variables.tf` — vpc_cidr, availability_zones, subnet CIDRs, environment, project_name
- `eusolicit-app/infra/terraform/modules/networking/outputs.tf` — vpc_id, private_subnet_ids, public_subnet_ids, nat_gateway_id
- `eusolicit-app/infra/terraform/modules/kubernetes/main.tf` — EKS/node group/OIDC placeholders
- `eusolicit-app/infra/terraform/modules/kubernetes/variables.tf` — cluster_name, cluster_version, vpc_id, subnet_ids, node sizing, environment
- `eusolicit-app/infra/terraform/modules/kubernetes/outputs.tf` — cluster_endpoint, cluster_name, cluster_certificate_authority, oidc_provider_arn
- `eusolicit-app/infra/terraform/modules/database/main.tf` — RDS PostgreSQL 16 placeholders
- `eusolicit-app/infra/terraform/modules/database/variables.tf` — instance_class, allocated_storage, db_name, engine_version (16), vpc_id, subnet_ids, multi_az, backup_retention_period, environment
- `eusolicit-app/infra/terraform/modules/database/outputs.tf` — db_endpoint, db_port, db_name, db_instance_id
- `eusolicit-app/infra/terraform/modules/redis/main.tf` — ElastiCache Redis 7 placeholders
- `eusolicit-app/infra/terraform/modules/redis/variables.tf` — node_type, engine_version (7.0), num_cache_nodes, vpc_id, subnet_ids, environment
- `eusolicit-app/infra/terraform/modules/redis/outputs.tf` — redis_endpoint, redis_port, redis_cluster_id
- `eusolicit-app/infra/terraform/modules/storage/main.tf` — S3 bucket placeholders (documents, backups)
- `eusolicit-app/infra/terraform/modules/storage/variables.tf` — bucket_prefix, enable_versioning, lifecycle_rules, environment
- `eusolicit-app/infra/terraform/modules/storage/outputs.tf` — documents_bucket_arn, documents_bucket_name, backups_bucket_arn
- `eusolicit-app/infra/terraform/modules/monitoring/main.tf` — CloudWatch/AMP/AMG placeholders
- `eusolicit-app/infra/terraform/modules/monitoring/variables.tf` — log_retention_days, alarm_email, enable_prometheus, enable_grafana, environment
- `eusolicit-app/infra/terraform/modules/monitoring/outputs.tf` — log_group_arns, sns_topic_arn, prometheus_endpoint, grafana_endpoint
- `eusolicit-app/infra/terraform/environments/dev/main.tf` — dev environment module calls (db.t3.micro, cache.t3.micro, 2 nodes, single-AZ)
- `eusolicit-app/infra/terraform/environments/dev/terraform.tfvars` — dev variable overrides
- `eusolicit-app/infra/terraform/environments/staging/main.tf` — staging environment module calls (db.t3.small, cache.t3.small, 3 nodes, multi-AZ)
- `eusolicit-app/infra/terraform/environments/staging/terraform.tfvars` — staging variable overrides
- `eusolicit-app/infra/terraform/environments/prod/main.tf` — prod environment module calls (db.r6g.large, cache.r6g.large, 5+ nodes, multi-AZ, HA)
- `eusolicit-app/infra/terraform/environments/prod/terraform.tfvars` — prod variable overrides

Modified files:
- `eusolicit-app/infra/terraform/README.md` — REPLACED placeholder with comprehensive documentation (module structure, usage, deployment commands, state management)
- `eusolicit-app/.gitignore` — added terraform artifact entries (.terraform/, *.tfstate, etc.)

## Change Log

- 2026-04-06: Story 1-10 implemented — complete Terraform scaffold with 6 root files, 6 modules (18 files), 3 environments (6 files), comprehensive README, terraform validate passes, 142/142 ATDD tests pass
