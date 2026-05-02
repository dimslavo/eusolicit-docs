# Epic 1: Core Platform & User Foundation

This epic establishes the foundational SaaS capabilities, allowing users to register, manage their accounts and company profiles, and subscribe to paid plans. It is the bedrock upon which all other features are built.

**FRs covered:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-6, FR-7, FR-10, FR-11, FR-12, FR-13, FR-14

## Stories

### Story 1.1: User Registration with Email and Password
As a new user,
I want to register for an account using my email and password,
So that I can access the platform's features.

**Acceptance Criteria:**
**Given** I am on the registration page
**When** I enter a valid, unique email address and a password that meets the strength requirements
**Then** a new user account is created in the system
**And** I am sent a verification email to the provided address.

### Story 1.2: User Login with Email and Password
As a registered user,
I want to log in to my account using my email and password,
So that I can access my workspace and data.

**Acceptance Criteria:**
**Given** I am a registered user with a verified email
**When** I enter my correct email and password on the login page
**Then** I am authenticated and redirected to my dashboard
**And** a secure session (JWT) is created.

### Story 1.3: User Email Verification
As a new user who has just registered,
I want to verify my email address by clicking a link in an email,
So that I can secure my account and unlock core platform features.

**Acceptance Criteria:**
**Given** I have registered an account but not yet verified my email
**When** I click the unique verification link in the email sent to me
**Then** my email address is marked as verified in the system
**And** I am redirected to a confirmation page or my dashboard.

### Story 1.4: User Registration and Login with Google
As a new or existing user,
I want to register or log in using my Google account,
So that I can access the platform without creating a separate password.

**Acceptance Criteria:**
**Given** I am on the registration or login page
**When** I click the "Sign in with Google" button and complete the Google authentication flow
**Then** a new account is created if one doesn't exist for that Google email, or I am logged into my existing account
**And** I am redirected to my dashboard.

### Story 1.5: Company Profile Management
As a company admin,
I want to create and update my company's profile with legal and business information,
So that this data can be used to auto-fill documents and personalize AI features.

**Acceptance Criteria:**
**Given** I am logged in as an admin for my company
**When** I navigate to the company profile page
**Then** I can fill in and save fields such as company name, VAT number, address, and sectors of expertise
**And** the saved information is stored securely against my company's workspace.

### Story 1.6: User Invitation and Role Management
As a company admin,
I want to invite team members to my company workspace and assign them roles,
So that my team can collaborate on the platform with appropriate permissions.

**Acceptance Criteria:**
**Given** I am an admin in my company's workspace
**When** I enter a team member's email address and select a role (e.g., Bid Manager, Contributor)
**Then** an invitation email is sent to the team member
**And** upon accepting the invitation, they are added to the workspace with the assigned role.

### Story 1.7: Subscription with Stripe (Starter and Professional)
As a user,
I want to subscribe to a paid plan (Starter or Professional) using my credit card,
So that I can access premium features.

**Acceptance Criteria:**
**Given** I am on the billing or upgrade page
**When** I select a plan and enter valid credit card details via the Stripe checkout form
**Then** a subscription is created in Stripe and my account is upgraded to the selected tier
**And** I receive a confirmation of my subscription.

### Story 1.8: 14-day Free Trial for Professional Tier
As a new user,
I want to be automatically enrolled in a 14-day free trial of the Professional tier upon registration,
So that I can experience the full value of the platform before committing to a paid plan.

**Acceptance Criteria:**
**Given** I have just registered a new account
**When** I log in for the first time
**Then** my account has access to all Professional tier features
**And** I can see a visible indicator of how many days are left in my trial.

### Story 1.9: Feature Gating Based on Subscription
As a user on a specific subscription tier,
I want to only see and access the features included in my plan,
So that the user interface is clean and I understand the value of upgrading.

**Acceptance Criteria:**
**Given** I am logged in with a 'Starter' tier subscription
**When** I attempt to access a 'Professional' tier feature (e.g., advanced collaboration)
**Then** I am shown a message explaining the feature is not in my plan and a prompt to upgrade
**And** the UI for the gated feature is visually distinct (e.g., disabled with a lock icon).

### Story 1.10: Self-service Subscription Management
As a paying user,
I want to be able to manage my subscription (upgrade, downgrade, cancel) through a self-service portal,
So that I have control over my billing without needing to contact support.

**Acceptance Criteria:**
**Given** I am a subscribed user
**When** I click the "Manage Subscription" button in my account settings
**Then** I am securely redirected to a Stripe customer portal where I can view my invoices and modify my subscription.
