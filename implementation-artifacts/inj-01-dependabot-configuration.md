# INJ-01: Dependabot Configuration
**Status:** ready-for-dev
**Epic:** 13
**Priority:** High

As a platform operator,
I want Dependabot configured for all Python and JavaScript packages,
so that known CVEs in any dependency are automatically surfaced as PRs.

## Acceptance Criteria:
1. `.github/dependabot.yml` created with:
   - Python pip: weekly schedule, all 5 services + packages
   - npm/pnpm: weekly schedule, all frontend apps and packages
   - GitHub Actions: monthly schedule
2. File merged to main branch (no open PRs required, only config committed)
3. First Dependabot scan initiated (no failures required, only activation)

## Tasks:
- [ ] Create .github/dependabot.yml
- [ ] Configure pip ecosystem for services/client-api, admin-api, data-pipeline, ai-gateway, notification
- [ ] Configure npm ecosystem for frontend/apps/client, frontend/apps/admin, frontend/packages/ui
- [ ] PR + merge
