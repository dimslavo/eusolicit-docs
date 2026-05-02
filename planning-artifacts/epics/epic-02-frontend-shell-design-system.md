# Epic 2: Frontend Shell & Design System

Developers have a robust monorepo setup providing the Next.js shell, localized content, and shared UI components mapping to UX specifications for users.

### Story 2.1: Turborepo Bootstrap
As a developer,
I want a monorepo with shared UI and config,
So that I can efficiently build and maintain the client and admin applications.

**Acceptance Criteria:**
**Given** a new development environment
**When** I run the Turborepo build pipeline
**Then** apps/client and apps/admin build successfully using shared packages/ui and packages/config
**And** shadcn/ui components are available and accessible via keyboard navigation with visible focus rings

### Story 2.2: QueryGuard, useZodForm, FormField
As a feature dev,
I want consistent fetching and forms,
So that the application has a unified data handling and submission architecture.

**Acceptance Criteria:**
**Given** I am building a feature
**When** I implement data fetching and forms
**Then** all remote state is wrapped in a <QueryGuard> component
**And** forms use useZodForm with <FormField> wrappers, and response types are generated via openapi-typescript

### Story 2.3: i18n & no-literal-text Lint
As a BG/EN customer,
I want fully localized UI,
So that I can use the platform in my native language.

**Acceptance Criteria:**
**Given** the application UI
**When** strings are rendered
**Then** all user-visible strings are keyed in next-intl
**And** pnpm check:i18n is a CI release gate and no-literal-text rule is active
