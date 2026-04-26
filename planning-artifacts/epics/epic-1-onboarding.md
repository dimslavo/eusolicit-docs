# Epic 1: Onboarding & Account Management

**Goal:** Users can register, select subscription tiers, and manage company roles and audit logs securely.
**FRs covered:** FR11, FR13, FR14

## Stories

### Story 1.1: Project Bootstrap and Architecture Setup
As a developer,
I want to set up the initial project from the starter template,
So that I have the foundational architecture for all future stories.

**Acceptance Criteria:**
**Given** I am initializing the application
**When** I configure the monorepo
**Then** I must set up Next.js 14 App Router, TypeScript, Zustand, TanStack Query v5, and shadcn/ui for the frontend
**And** Python 3.12+, FastAPI, PostgreSQL 16, and Redis 7 for the backend
**And** establish the base schemas (client, admin, pipeline, ai, notification, shared)

### Story 1.2: User Registration and Identity
As a new user,
I want to securely register for an account using my email and password,
So that I can access the platform and configure my company profile.

**Acceptance Criteria:**
**Given** I am on the registration page
**When** I submit my email and a password
**Then** the password must be bounded at max_length=128 and hashed asynchronously
**And** inline validation (Zod) must provide errors onBlur
**And** my session must be protected by `<AuthGuard>` and `middleware.ts` cookie checks

### Story 1.3: Subscription Tier Selection with Stripe
As a registered user,
I want to select or upgrade my subscription tier (Free, Starter, Pro, Enterprise),
So that I can access features like AI proposal generation.

**Acceptance Criteria:**
**Given** I am on the billing page
**When** I choose to upgrade to Professional and complete the Stripe checkout
**Then** the webhook processor must update my tier using idempotent Redis DB constraints (`SETNX`)
**And** the API must instantly apply access rights

### Story 1.4: Role-Based Access Control (RBAC)
As a company admin,
I want to manage roles for my team members,
So that I can control access to bid creation and account settings.

**Acceptance Criteria:**
**Given** I am a company admin
**When** I navigate to the team settings
**Then** I can assign roles (admin, bid_manager, contributor, reviewer, read_only)
**And** the system must enforce company role ceilings and entity-level overrides

### Story 1.5: Immutable Audit Logging
As a platform admin,
I want the system to automatically log critical actions,
So that I have an immutable trail of security and access events.

**Acceptance Criteria:**
**Given** a user performs a POST, PATCH, or DELETE action
**When** the request is processed
**Then** the system must append an entry to the `shared.audit_log` with user actions, IP, and before/after values