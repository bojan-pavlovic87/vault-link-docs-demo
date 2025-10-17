# VaultLink Branching Strategy

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Client:** Synertec  
**Project:** VaultLink MVP AWS Rebuild  
**Purpose:** Git Workflow & Branching Guidelines

---

## Executive Summary

This document defines the Git branching strategy for VaultLink MVP development. The strategy balances team collaboration, code quality, deployment safety, and release management using a trunk-based development approach with environment-specific branches.

**Key Principles:**
- Trunk-based development with short-lived feature branches
- Environment promotion workflow (dev  staging  production)
- Automated CI/CD pipeline triggered by branch merges
- Protected branches with required reviews
- Semantic versioning for releases

---

## Branch Structure

### Main Branches (Long-Lived)

#### `main`
- **Purpose:** Production-ready code
- **Deployment:** Automatically deploys to production environment
- **Protection Rules:**
  - Requires 2 approvals from code owners
  - All status checks must pass (tests, security scans, build)
  - No direct commits allowed
  - Enforce linear history (rebase/squash merges)
  - Signed commits required
- **Merge Sources:** Only from `release/*` branches
- **Tagging:** All production releases tagged with semantic versioning (e.g., `v1.0.0`)

#### `develop`
- **Purpose:** Integration branch for ongoing development
- **Deployment:** Automatically deploys to dev environment
- **Protection Rules:**
  - Requires 1 approval
  - All tests must pass
  - No direct commits allowed
  - Up-to-date branch check enabled
- **Merge Sources:** Feature branches, bugfix branches, hotfix branches
- **Versioning:** Tagged with pre-release versions (e.g., `v1.0.0-dev.1`)

#### `staging`
- **Purpose:** Pre-production validation and UAT
- **Deployment:** Automatically deploys to staging environment
- **Protection Rules:**
  - Requires 1 approval from PO or Tech Lead
  - All tests + E2E suite must pass
  - Security scans must be clean
  - Performance tests must meet benchmarks
- **Merge Sources:** Only from `develop` branch
- **Versioning:** Tagged with release candidates (e.g., `v1.0.0-rc.1`)

---

### Supporting Branches (Short-Lived)

#### `feature/*`
- **Purpose:** New feature development
- **Naming Convention:** `feature/epic-number-brief-description`
  - Examples:
    - `feature/3-anonymous-statement-access`
    - `feature/4-stripe-payment-integration`
    - `feature/9-angular-responsive-ui`
- **Lifecycle:** Created from `develop`, merged back to `develop`
- **Lifespan:** Maximum 5 working days (encourage small, incremental changes)
- **CI/CD:** Runs unit tests, linting, security scans on every push
- **Deletion:** Automatically deleted after merge

**Workflow:**
```bash
# Create feature branch
git checkout develop
git pull origin develop
git checkout -b feature/3-qr-code-scanning

# Work on feature (commit frequently)
git add .
git commit -m "feat: implement QR code scanner component"

# Keep up-to-date with develop
git fetch origin
git rebase origin/develop

# Push and create PR
git push origin feature/3-qr-code-scanning
```

#### `bugfix/*`
- **Purpose:** Non-critical bug fixes during development
- **Naming Convention:** `bugfix/issue-number-brief-description`
  - Examples:
    - `bugfix/123-statement-display-alignment`
    - `bugfix/124-payment-validation-error`
- **Lifecycle:** Created from `develop`, merged back to `develop`
- **Lifespan:** Maximum 2 working days
- **CI/CD:** Same as feature branches

#### `hotfix/*`
- **Purpose:** Critical production bug fixes
- **Naming Convention:** `hotfix/issue-number-brief-description`
  - Examples:
    - `hotfix/567-payment-processing-failure`
    - `hotfix/568-security-vulnerability-cve-2025-xxxx`
- **Lifecycle:** 
  - Created from `main`
  - Merged to both `main` AND `develop` (to keep branches in sync)
- **Lifespan:** Immediate (hours, not days)
- **CI/CD:** Full test suite + security scans
- **Versioning:** Increments patch version (e.g., `v1.0.0`  `v1.0.1`)

**Hotfix Workflow:**
```bash
# Create hotfix from main
git checkout main
git pull origin main
git checkout -b hotfix/567-payment-processing-failure

# Fix the issue
git add .
git commit -m "fix: resolve payment processing timeout issue"

# Test thoroughly
# Run full test suite locally

# Push and create PR to main
git push origin hotfix/567-payment-processing-failure

# After merge to main, also merge to develop
git checkout develop
git pull origin develop
git merge hotfix/567-payment-processing-failure
git push origin develop
```

#### `release/*`
- **Purpose:** Release preparation, final bug fixes, version bumping
- **Naming Convention:** `release/version-number`
  - Examples:
    - `release/1.0.0`
    - `release/1.1.0`
- **Lifecycle:**
  - Created from `staging` after UAT approval
  - Merged to `main` for production deployment
  - Also merged back to `develop` to sync changes
- **Activities:**
  - Version number updates
  - CHANGELOG updates
  - Final documentation updates
  - Minor bug fixes only (no new features)
- **Lifespan:** 1-3 days maximum

**Release Workflow:**
```bash
# Create release branch from staging
git checkout staging
git pull origin staging
git checkout -b release/1.0.0

# Update version numbers
# Update CHANGELOG.md
# Update deployment documentation

git add .
git commit -m "chore: prepare release 1.0.0"

# Push and create PR to main
git push origin release/1.0.0

# After merge to main, tag the release
git checkout main
git pull origin main
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# Merge back to develop
git checkout develop
git pull origin develop
git merge release/1.0.0
git push origin develop
```

#### `experimental/*` (Optional)
- **Purpose:** Proof-of-concept work, spike investigations
- **Naming Convention:** `experimental/spike-description`
  - Examples:
    - `experimental/dynamodb-stream-processing`
    - `experimental/lambda-performance-optimization`
- **Lifecycle:** Created from `develop`, may or may not be merged
- **Lifespan:** Time-boxed (usually 1-3 days)
- **CI/CD:** Optional (may skip some checks)
- **Deletion:** Deleted after POC conclusion (document findings first)

---

## Commit Message Convention

Following **Conventional Commits** specification for automated changelog generation and semantic versioning.

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- **feat:** New feature (triggers minor version bump)
- **fix:** Bug fix (triggers patch version bump)
- **docs:** Documentation changes only
- **style:** Code style changes (formatting, no logic change)
- **refactor:** Code refactoring (no feature change or bug fix)
- **perf:** Performance improvements
- **test:** Adding or updating tests
- **build:** Build system or dependency changes
- **ci:** CI/CD pipeline changes
- **chore:** Other changes (version bumps, releases)
- **revert:** Reverting previous commits

### Scopes (Optional)

- `frontend` - Angular application
- `backend` - .NET Lambda functions
- `infrastructure` - CDK/CloudFormation
- `auth` - Authentication/authorization
- `payment` - Stripe integration
- `prism` - Prism integration
- `docs` - Documentation
- `tests` - Testing

### Examples

```bash
feat(payment): implement Stripe Connect payment intent creation

- Add PaymentService with createPaymentIntent method
- Integrate Stripe Elements for secure card input
- Add payment confirmation webhook handler

Closes #123

---

fix(frontend): resolve statement aging calculation error

The aging buckets were incorrectly calculated for invoices
with dates in the previous calendar year.

Fixes #456

---

docs(readme): update local development setup instructions

---

chore(release): bump version to 1.0.0
```

### Breaking Changes

For breaking changes, add `BREAKING CHANGE:` in the footer:

```bash
feat(api)!: change payment API response structure

BREAKING CHANGE: Payment API now returns nested object structure
instead of flat response. Clients must update to new schema.

Migration guide: docs/api-migration-v2.md
```

---

## Pull Request Workflow

### Creating a Pull Request

**Title Format:**
```
<type>(<scope>): <brief description>
```

**Description Template:**
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Feature (new functionality)
- [ ] Bug fix (non-breaking fix)
- [ ] Hotfix (critical production fix)
- [ ] Documentation update
- [ ] Refactoring
- [ ] Performance improvement

## Related Issues
Closes #123
Related to #456

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] E2E tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] All tests passing locally
- [ ] Security scan passing

## Screenshots (if applicable)
[Add screenshots for UI changes]

## Deployment Notes
[Any special deployment considerations]
```

### Review Process

**Code Review Requirements:**

| Target Branch | Required Approvals | Required Checks |
|---------------|-------------------|-----------------|
| `develop` | 1 (any developer) | Unit tests, linting, SAST |
| `staging` | 1 (Tech Lead/PO) | All tests, E2E suite, security |
| `main` | 2 (Tech Lead + Senior Dev) | All checks, manual approval |

**Review Criteria:**
- Code quality and readability
- Test coverage (>80% for new code)
- Security considerations
- Performance implications
- Documentation completeness
- Error handling
- Logging appropriateness

**Review Timeline:**
- Feature/bugfix PRs: Review within 24 hours
- Hotfix PRs: Review within 2 hours (critical)
- Release PRs: Review within 4 hours

### PR Status Checks

All PRs must pass:

1. **Build & Compile**
   - Angular build succeeds
   - .NET build succeeds
   - No compilation errors

2. **Unit Tests**
   - Frontend tests pass (Jasmine/Karma)
   - Backend tests pass (xUnit)
   - Code coverage 80% (new code)

3. **Linting & Formatting**
   - ESLint (Angular)
   - Prettier (Angular)
   - .NET analyzer rules

4. **Security Scans**
   - SonarQube SAST scan
   - Dependency vulnerability scan (Snyk)
   - No critical/high vulnerabilities

5. **Integration Tests** (for `staging` and `main`)
   - API integration tests pass
   - DynamoDB integration tests pass
   - SQS integration tests pass

6. **E2E Tests** (for `staging` and `main`)
   - Critical user journeys pass
   - Payment flow validated
   - Query submission validated

---

## Merge Strategies

### Feature  Develop
- **Strategy:** Squash and merge
- **Rationale:** Clean commit history, single commit per feature
- **Commit Message:** PR title becomes commit message

### Develop  Staging
- **Strategy:** Merge commit (no squash)
- **Rationale:** Preserve feature commit history for UAT traceability

### Release  Main
- **Strategy:** Merge commit with tag
- **Rationale:** Preserve release history, enable rollback

### Hotfix  Main
- **Strategy:** Merge commit
- **Rationale:** Clear audit trail for production fixes

---

## Version Tagging

Following **Semantic Versioning (SemVer)**: `MAJOR.MINOR.PATCH`

### Version Numbers

- **MAJOR:** Breaking changes (e.g., API contract changes)
- **MINOR:** New features (backward compatible)
- **PATCH:** Bug fixes (backward compatible)

### Tag Format

```bash
# Production releases
v1.0.0
v1.1.0
v1.1.1

# Release candidates (staging)
v1.0.0-rc.1
v1.0.0-rc.2

# Development builds
v1.0.0-dev.1
v1.0.0-dev.2
```

### Tagging Workflow

```bash
# Tag main branch after release merge
git checkout main
git pull origin main

# Create annotated tag
git tag -a v1.0.0 -m "Release 1.0.0 - MVP Launch

Features:
- Anonymous statement access
- Stripe Connect payment processing
- Prism integration
- Multi-tenant white-labeling

See CHANGELOG.md for full details"

# Push tag
git push origin v1.0.0

# Tag triggers production deployment via CI/CD
```

---

## Environment Promotion Flow

```

   feature    
   
                  > 
          develop    > Dev Environment
   bugfix                   
                                  
                                                 
                                        
                                           staging    > Staging Environment
                                                  
                                                                
                                                           (UAT Testing)
                                                                
                                                  
                                          release/*   
                                        
                                                 
                                                 
                                        
                                             main     > Production
                                        
                                                 
                                                 
                                             (Tagged)
                                            v1.0.0
```

### Promotion Process

**Week 1-2: Feature Development**
```bash
feature/*  develop (daily merges)
develop deploys to dev environment automatically
```

**Week 3: Staging Promotion**
```bash
develop  staging (Friday EOD)
staging deploys to staging environment
UAT begins following week
```

**Week 4: UAT & Release Prep**
```bash
UAT testing on staging environment
Bug fixes: bugfix/*  develop  staging
Release branch created: release/1.0.0
```

**Week 5: Production Release**
```bash
release/1.0.0  main (after PO approval)
main tagged with v1.0.0
Production deployment (blue-green)
release/1.0.0  develop (sync)
```

---

## Branch Naming Conventions Summary

| Branch Type | Pattern | Example |
|-------------|---------|---------|
| Feature | `feature/<epic>-<description>` | `feature/3-anonymous-access` |
| Bugfix | `bugfix/<issue>-<description>` | `bugfix/123-payment-error` |
| Hotfix | `hotfix/<issue>-<description>` | `hotfix/567-security-patch` |
| Release | `release/<version>` | `release/1.0.0` |
| Experimental | `experimental/<spike>` | `experimental/lambda-perf` |

**Naming Rules:**
- Use lowercase
- Use hyphens (not underscores)
- Keep descriptions brief (3-5 words max)
- Include issue/epic number when applicable
- Be descriptive but concise

---

## Repository Structure

### Monorepo vs. Multi-Repo

**Recommendation:** **Multi-repository approach**

#### Repositories

**1. vaultlink-infrastructure**
- **Purpose:** CDK/CloudFormation IaC code
- **Language:** TypeScript (CDK) or YAML (CloudFormation)
- **Branches:** Same strategy as main project
- **Deployment:** Infrastructure changes deploy before application code

**2. vaultlink-backend**
- **Purpose:** .NET Lambda functions, APIs
- **Language:** C# / .NET 8
- **Branches:** Full branching strategy
- **Deployment:** Lambda functions deployed per environment

**3. vaultlink-frontend**
- **Purpose:** Angular SPA
- **Language:** TypeScript / Angular 20+
- **Branches:** Full branching strategy
- **Deployment:** S3 + CloudFront per environment

**4. vaultlink-docs** (current repo)
- **Purpose:** Project documentation
- **Language:** Markdown
- **Branches:** Simplified (main + feature/*)
- **Deployment:** GitHub Pages or S3 static hosting

### Why Multi-Repo?

 **Pros:**
- Independent versioning per component
- Separate CI/CD pipelines (faster builds)
- Team ownership boundaries (frontend vs backend)
- Smaller repository size
- Infrastructure changes isolated from application

 **Cons:**
- Cross-repo dependency management
- Coordinating releases across repos
- Multiple PR processes

**Mitigation:**
- Use unified release orchestration
- Shared CI/CD templates
- Cross-repo tagging for releases

---

## CI/CD Integration

### GitHub Actions Workflows

**Feature Branch Pipeline:**
```yaml
name: Feature Branch CI
on:
  pull_request:
    branches: [develop]
    types: [opened, synchronize]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
      - name: Lint code
      - name: Run unit tests
      - name: Check code coverage (80%)
      - name: SAST scan (SonarQube)
      - name: Dependency scan (Snyk)
      - name: Build artifacts
```

**Develop Branch Pipeline:**
```yaml
name: Deploy to Dev
on:
  push:
    branches: [develop]

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run full test suite
      - name: Build artifacts
      - name: Deploy to dev environment (CDK deploy)
      - name: Run integration tests
      - name: Notify team (Slack)
```

**Staging Branch Pipeline:**
```yaml
name: Deploy to Staging
on:
  push:
    branches: [staging]

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run full test suite
      - name: Run E2E tests
      - name: Performance smoke tests
      - name: DAST security scan
      - name: Build artifacts
      - name: Deploy to staging (CDK deploy)
      - name: Run post-deployment tests
      - name: Notify PO for UAT
```

**Production Pipeline:**
```yaml
name: Deploy to Production
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  production-deployment:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - name: Run full test suite
      - name: Security validation
      - name: Build production artifacts
      - name: Blue-green deployment
      - name: Deploy to 10% (canary)
      - name: Monitor metrics (15 min)
      - name: Deploy to 100% or rollback
      - name: Run smoke tests
      - name: Update CHANGELOG
      - name: Create GitHub release
      - name: Notify stakeholders
```

---

## Conflict Resolution

### Merge Conflicts

**Prevention:**
- Keep feature branches short-lived (<5 days)
- Rebase frequently from `develop`
- Clear ownership of files/modules

**Resolution Process:**
1. **Pull latest develop:**
   ```bash
   git checkout feature/my-branch
   git fetch origin
   git rebase origin/develop
   ```

2. **Resolve conflicts:**
   - Open conflicted files
   - Review both changes
   - Choose correct version or combine
   - Test thoroughly after resolution

3. **Continue rebase:**
   ```bash
   git add .
   git rebase --continue
   ```

4. **Force push (if already pushed):**
   ```bash
   git push origin feature/my-branch --force-with-lease
   ```

### Conflict Ownership

- Feature owner resolves conflicts in their branch
- For complex conflicts, pair with code owner
- Document resolution decision in PR comments

---

## Best Practices

### DO 

-  Commit frequently with meaningful messages
-  Keep feature branches small and focused
-  Rebase onto develop daily
-  Write descriptive PR descriptions
-  Review PRs promptly (within 24 hours)
-  Delete branches after merge
-  Tag all production releases
-  Update CHANGELOG with every release
-  Run tests locally before pushing
-  Use conventional commit format

### DON'T 

-  Commit directly to `main`, `develop`, or `staging`
-  Force push to shared branches
-  Merge PRs without approval
-  Include unrelated changes in PRs
-  Leave stale branches (>2 weeks)
-  Commit secrets or credentials
-  Skip CI/CD checks
-  Merge broken code to develop
-  Create mega-PRs (>500 lines)
-  Bypass required reviews

---

## Rollback Procedures

### Staging/Dev Rollback

```bash
# Revert last commit
git checkout develop
git revert HEAD
git push origin develop

# Or revert specific commit
git revert <commit-hash>
git push origin develop
```

### Production Rollback

**Method 1: Revert Commit**
```bash
git checkout main
git revert <bad-commit-hash>
git push origin main
# Tag reverted version
git tag -a v1.0.1 -m "Rollback of v1.0.0"
git push origin v1.0.1
```

**Method 2: Redeploy Previous Tag**
```bash
# Trigger deployment of previous version
git checkout v1.0.0
# Deploy via CI/CD or manual CDK deploy
```

**Method 3: Blue-Green Instant Rollback**
```bash
# Switch traffic back to blue environment
aws lambda update-alias \
  --function-name VaultLink-API \
  --name production \
  --routing-config AdditionalVersionWeights="{}"
```

---

## Branch Lifecycle Management

### Stale Branch Cleanup

**Automated Cleanup Policy:**
- Feature branches: Delete after merge
- Bugfix branches: Delete after merge
- Merged branches: Delete after 7 days
- Unmerged feature branches: Alert after 14 days, delete after 30 days

**Manual Cleanup:**
```bash
# List merged branches
git branch --merged develop

# Delete local merged branches
git branch --merged develop | grep -v "develop\|main\|staging" | xargs git branch -d

# Delete remote merged branches
git push origin --delete feature/old-feature
```

### Protected Branch Settings

**GitHub Branch Protection Rules:**

**`main`:**
- Require pull request reviews (2 approvals)
- Dismiss stale reviews
- Require review from code owners
- Require status checks to pass
- Require branches to be up to date
- Require signed commits
- Require linear history
- Restrict who can push (admins only)

**`develop`:**
- Require pull request reviews (1 approval)
- Require status checks to pass
- Require branches to be up to date

**`staging`:**
- Require pull request reviews (1 approval from PO/Tech Lead)
- Require status checks to pass
- Include E2E tests in required checks

---

## Team Workflows

### Daily Development

**Morning:**
1. Pull latest `develop`
2. Rebase feature branch if needed
3. Continue work on feature

**Throughout Day:**
4. Commit frequently (every logical change)
5. Push to remote at least once per day

**End of Day:**
6. Push all commits
7. Open draft PR if feature incomplete (for visibility)

### Sprint Workflow

**Monday (Sprint Start):**
- Create feature branches from develop
- Set up local development environment

**Tuesday-Thursday:**
- Active development
- Daily stand-ups discuss PR status
- Code reviews

**Friday:**
- Merge completed features to develop
- Deploy to dev environment
- Demo to PO (if ready)

**End of Sprint:**
- Merge develop  staging
- Begin UAT

### Release Workflow

**T-7 days: Code Freeze**
- Last feature merges to develop
- Only bug fixes allowed

**T-5 days: Staging Deployment**
- Merge develop  staging
- UAT begins

**T-3 days: Release Branch**
- Create release branch
- Final testing and fixes

**T-1 day: Production Prep**
- Final approval from PO
- Merge release  main
- Schedule deployment

**T-0: Production Deployment**
- Tag version
- Blue-green deployment
- Monitor for 24 hours

---

## Documentation

### Required Documentation Updates
You still can't hear me. Great OK. So branching strategy in a sense. We need to understand one thing in here that is a question. Do we want to? These are basic questions. In a smaller sense. So do we want to deal with a mono repo or we want to build? 
When merging to `main`, ensure:

- [ ] CHANGELOG.md updated with version changes
- [ ] API documentation updated (if API changes)
- [ ] README.md updated (if setup changes)
- [ ] Migration guides written (if breaking changes)
- [ ] Release notes drafted (for stakeholders)

---

## Tools & Integrations

### Recommended Tools

- **Git Client:** Git CLI, GitHub Desktop, or SourceTree
- **Code Review:** GitHub Pull Requests
- **CI/CD:** GitHub Actions or AWS CodePipeline
- **Branch Protection:** GitHub branch rules
- **Commit Enforcement:** Husky + commitlint
- **Changelog Generation:** conventional-changelog
- **Version Bumping:** standard-version or semantic-release

### Git Hooks (Husky)

**Pre-commit:**
```bash
# .husky/pre-commit
npm run lint
npm run format
npm run test:unit
```

**Commit-msg:**
```bash
# .husky/commit-msg
npx commitlint --edit $1
```

**Pre-push:**
```bash
# .husky/pre-push
npm run test:all
```

---

## FAQ

**Q: Can I commit directly to develop?**
A: No. All changes must go through PR process.

**Q: What if my feature branch is behind develop?**
A: Rebase your branch: `git rebase origin/develop`

**Q: How do I handle a hotfix?**
A: Create from `main`, merge to both `main` and `develop`.

**Q: Can I merge my own PR?**
A: No. All PRs require approval from another team member.

**Q: What if CI/CD checks fail?**
A: Fix issues locally, push updates, checks will re-run.

**Q: How long should PRs stay open?**
A: Aim for <24 hour turnaround on reviews.

**Q: What if I accidentally commit to main?**
A: Contact Tech Lead immediately. May require force push (risky).

**Q: Should I rebase or merge?**
A: Rebase feature branches onto develop. Merge commits for environment promotions.

---

## Success Metrics

**Branch Strategy KPIs:**

- Average feature branch lifespan: <5 days
- PR review time: <24 hours
- Merge conflict rate: <5%
- Hotfix frequency: <1 per month
- Failed deployments: <2%
- Rollback frequency: <1 per quarter
- Code review participation: 100% of team

---

**For MVP scope, see `brief.md`**  
**For infrastructure details, see `aws-infrastructure-services.md`**  
**For testing approach, see `testing-strategy.md`**  
**For epic breakdown, see `epics.md`**

---

**Document End**
