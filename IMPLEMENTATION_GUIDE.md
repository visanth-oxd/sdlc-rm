# Implementation Guide: Branching Strategy Setup

This guide provides step-by-step instructions for implementing the branching strategy across your organization.

**Architecture**: Each service has its own GitHub repository. Apply this setup process to each service repository independently.

## Prerequisites

- Git repository access (GitHub, GitLab, or Bitbucket)
- CI/CD platform access
- Admin access to repository settings
- Understanding of your current deployment infrastructure
- List of all service repositories that need to be configured

## Phase 0: Service Repository Inventory

Before starting, create an inventory of all service repositories:

```bash
# Example: List all service repositories
# services/
#   ├── user-service/
#   ├── payment-service/
#   ├── order-service/
#   ├── notification-service/
#   └── ... (100+ services)
```

**Action Items**:
- [ ] List all service repositories
- [ ] Identify service dependencies
- [ ] Prioritize services for initial rollout
- [ ] Document service-to-environment mappings

## Phase 1: Repository Setup (Per Service)

**Repeat these steps for each service repository:**

### Step 1: Create Release Branches

```bash
# Ensure main is up to date
git checkout main
git pull origin main

# Create beta release branch (n+1)
git checkout -b release/n+1
git push origin release/n+1

# Create alpha release branch (n+2)
git checkout -b release/n+2
git push origin release/n+2

# Return to main
git checkout main
```

### Step 2: Configure Branch Protection Rules

#### For GitHub:

1. Navigate to: `Settings` → `Branches` → `Branch protection rules`
2. Add rule for `main`:
   - Branch name pattern: `main`
   - Require pull request reviews: ✅ (2 approvals)
   - Require status checks: ✅ (select all CI/CD checks)
   - Require branches to be up to date: ✅
   - Restrict pushes: ✅
   - Require linear history: ✅ (squash/rebase only)

3. Add rule for `release/*`:
   - Branch name pattern: `release/*`
   - Require pull request reviews: ✅ (1 approval)
   - Require status checks: ✅ (select relevant checks)
   - Restrict pushes: ✅

#### For GitLab:

1. Navigate to: `Settings` → `Repository` → `Protected Branches`
2. Protect `main`:
   - Allowed to merge: Maintainers
   - Allowed to push: No one
   - Allowed to force push: No
   - Code owner approval required: Yes

3. Protect `release/*`:
   - Allowed to merge: Developers + Maintainers
   - Allowed to push: Maintainers only

## Phase 2: CI/CD Pipeline Configuration

### Step 1: Create Pipeline Configuration

Create or update your CI/CD configuration file (e.g., `.github/workflows/deploy.yml`, `.gitlab-ci.yml`):

```yaml
# Example GitHub Actions workflow
name: Deploy to Environments

on:
  push:
    branches:
      - main
      - 'release/**'
      - 'hotfix/**'
      - 'feature/**'

jobs:
  detect-environment:
    runs-on: ubuntu-latest
    outputs:
      branch-type: ${{ steps.branch.outputs.type }}
      environment-layer: ${{ steps.branch.outputs.layer }}
      release-type: ${{ steps.branch.outputs.release }}
    steps:
      - id: branch
        run: |
          BRANCH="${{ github.ref_name }}"
          if [[ "$BRANCH" == "main" ]]; then
            echo "type=stable" >> $GITHUB_OUTPUT
            echo "layer=bld" >> $GITHUB_OUTPUT
            echo "release=stable-01" >> $GITHUB_OUTPUT
          elif [[ "$BRANCH" =~ ^release/n\+1 ]]; then
            echo "type=beta" >> $GITHUB_OUTPUT
            echo "layer=bld" >> $GITHUB_OUTPUT
            echo "release=beta-01" >> $GITHUB_OUTPUT
          elif [[ "$BRANCH" =~ ^release/n\+2 ]]; then
            echo "type=alpha" >> $GITHUB_OUTPUT
            echo "layer=bld" >> $GITHUB_OUTPUT
            echo "release=alpha-01" >> $GITHUB_OUTPUT
          elif [[ "$BRANCH" =~ ^hotfix/ ]]; then
            echo "type=stable" >> $GITHUB_OUTPUT
            echo "layer=bld" >> $GITHUB_OUTPUT
            echo "release=stable-01" >> $GITHUB_OUTPUT
          elif [[ "$BRANCH" =~ ^feature/ ]]; then
            # Determine from base branch
            echo "type=beta" >> $GITHUB_OUTPUT
            echo "layer=bld" >> $GITHUB_OUTPUT
            echo "release=beta-01" >> $GITHUB_OUTPUT
          fi

  deploy-bld:
    needs: detect-environment
    if: needs.detect-environment.outputs.layer == 'bld'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to BLD
        run: |
          ENV="${ENVIRONMENT_LAYER^^}-${RELEASE_TYPE}"
          echo "Deploying to $ENV"
          # Your deployment script here
          # deploy.sh --environment $ENV --branch ${{ github.ref_name }}
        env:
          ENVIRONMENT_LAYER: ${{ needs.detect-environment.outputs.layer }}
          RELEASE_TYPE: ${{ needs.detect-environment.outputs.release }}
```

### Step 2: Environment Promotion Workflow

Create a workflow for promoting between environments:

```yaml
name: Promote Environment

on:
  workflow_dispatch:
    inputs:
      source_env:
        description: 'Source environment'
        required: true
        type: choice
        options:
          - BLD-stable-01
          - INT-stable-01
          - PRE-stable-01
          - BLD-beta-01
          - INT-beta-01
          - PRE-beta-01
          - BLD-alpha-01
          - INT-alpha-01
      target_env:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - INT-stable-01
          - PRE-stable-01
          - PRD-stable-01
          - INT-beta-01
          - PRE-beta-01
          - INT-alpha-01

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - name: Validate promotion path
        run: |
          # Add validation logic
          echo "Promoting from ${{ github.event.inputs.source_env }} to ${{ github.event.inputs.target_env }}"
      
      - name: Run tests
        run: |
          # Run environment-specific tests
      
      - name: Promote deployment
        run: |
          # Your promotion script
          # promote.sh --from ${{ github.event.inputs.source_env }} --to ${{ github.event.inputs.target_env }}
```

## Phase 3: Service-Specific Configuration

### Service Repository Structure

Since each service has its own GitHub repository, each repository should:

1. **Use the same CI/CD workflow template** (see `.github/workflows/deploy.yml.example`)
2. **Follow the same branching strategy** independently
3. **Configure service-specific environment variables** in repository settings
4. **Set up service-specific deployment targets**

### Standardizing Across Service Repositories

To ensure consistency across 100+ service repositories:

#### Option 1: Shared Workflow Template (Recommended)

Create a shared workflow template repository that all services can reference:

```yaml
# In a shared repository: .github/workflows/shared-deploy.yml
# Each service repository references this template

name: Deploy Service

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      environment:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy ${{ inputs.service-name }}
        run: |
          # Standardized deployment script
          ./deploy.sh --service ${{ inputs.service-name }} --env ${{ inputs.environment }}
```

Each service repository then calls this shared workflow:

```yaml
# In each service repository: .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, 'release/**', 'hotfix/**']

jobs:
  deploy:
    uses: your-org/shared-workflows/.github/workflows/shared-deploy.yml@main
    with:
      service-name: ${{ github.repository }}
      environment: ${{ needs.detect.outputs.target-env }}
    secrets: inherit
```

#### Option 2: Copy Template to Each Repository

Copy the workflow template (`.github/workflows/deploy.yml.example`) to each service repository and customize:

```bash
# Script to apply template to all service repositories
#!/bin/bash

SERVICES=(
  "user-service"
  "payment-service"
  "order-service"
  # ... add all services
)

for service in "${SERVICES[@]}"; do
  echo "Setting up $service..."
  
  # Clone repository
  git clone "https://github.com/your-org/$service.git"
  cd "$service"
  
  # Copy workflow template
  mkdir -p .github/workflows
  cp ../templates/deploy.yml .github/workflows/deploy.yml
  
  # Customize service name
  sed -i "s/SERVICE_NAME/$service/g" .github/workflows/deploy.yml
  
  # Commit and push
  git add .github/workflows/deploy.yml
  git commit -m "chore: add standardized deployment workflow"
  git push origin main
  
  cd ..
done
```

### Service-Specific Environment Configuration

Each service repository should have environment-specific secrets and variables:

**GitHub Repository Settings → Secrets and variables → Actions**:
- `SERVICE_NAME`: Name of the service (e.g., `user-service`)
- `DEPLOYMENT_KEY`: Service-specific deployment key
- `ENVIRONMENT_URLS`: Service-specific environment URLs
- Service-specific API keys, database credentials, etc.

### Cross-Service Coordination

For releases that span multiple services, use:

1. **Release Coordination Script**:
```bash
#!/bin/bash
# scripts/coordinated-release.sh

SERVICES=("user-service" "payment-service" "order-service")
RELEASE_BRANCH="release/n+1"

for service in "${SERVICES[@]}"; do
  echo "Processing $service..."
  cd "$service"
  git checkout "$RELEASE_BRANCH"
  git pull origin "$RELEASE_BRANCH"
  # Trigger deployment or create PR
  cd ..
done
```

2. **Service Dependency Graph**: Maintain a dependency graph to determine deployment order
3. **Release Dashboard**: Track which services are part of which release

## Phase 4: Automation Scripts

### Branch Creation Helper Script

Create a script to help developers create branches correctly with Jira IDs:

```bash
#!/bin/bash
# scripts/create-branch.sh

BRANCH_TYPE=$1
JIRA_ID=$2
DESCRIPTION=$3
BASE_BRANCH=$4

# Validate Jira ID format (CORENGC-xxxx)
if [[ ! "$JIRA_ID" =~ ^CORENGC-[0-9]+$ ]]; then
  echo "❌ Error: Invalid Jira ID format. Expected: CORENGC-xxxx"
  echo "Example: CORENGC-1234"
  exit 1
fi

case $BRANCH_TYPE in
  hotfix)
    BASE_BRANCH=${BASE_BRANCH:-main}
    if [ -z "$DESCRIPTION" ]; then
      BRANCH_NAME="hotfix/$JIRA_ID"
    else
      BRANCH_NAME="hotfix/$JIRA_ID-$DESCRIPTION"
    fi
    ;;
  feature)
    BASE_BRANCH=${BASE_BRANCH:-release/n+1}
    if [ -z "$DESCRIPTION" ]; then
      BRANCH_NAME="feature/$JIRA_ID"
    else
      BRANCH_NAME="feature/$JIRA_ID-$DESCRIPTION"
    fi
    ;;
  release)
    BRANCH_NAME="release/$DESCRIPTION"
    BASE_BRANCH=${BASE_BRANCH:-main}
    JIRA_ID=""  # Release branches don't use Jira IDs
    ;;
  *)
    echo "Usage: $0 {hotfix|feature|release} <jira-id> <description> [base-branch]"
    echo ""
    echo "Examples:"
    echo "  $0 hotfix CORENGC-1234 critical-bug-fix"
    echo "  $0 feature CORENGC-5678 user-authentication release/n+1"
    echo "  $0 release v1.3.0"
    exit 1
    ;;
esac

git checkout $BASE_BRANCH
git pull origin $BASE_BRANCH
git checkout -b $BRANCH_NAME
echo "✅ Created branch: $BRANCH_NAME from $BASE_BRANCH"
if [ -n "$JIRA_ID" ]; then
  echo "📋 Jira ticket: $JIRA_ID"
fi
```

### Hotfix Cherry-pick Script

```bash
#!/bin/bash
# scripts/cherry-pick-hotfix.sh

COMMIT_HASH=$1

if [ -z "$COMMIT_HASH" ]; then
  echo "Usage: $0 <commit-hash>"
  exit 1
fi

# Cherry-pick to release/n+1
git checkout release/n+1
git pull origin release/n+1
git cherry-pick $COMMIT_HASH
git push origin release/n+1

# Cherry-pick to release/n+2
git checkout release/n+2
git pull origin release/n+2
git cherry-pick $COMMIT_HASH
git push origin release/n+2

# Return to main
git checkout main
echo "Cherry-picked $COMMIT_HASH to all release branches"
```

## Phase 5: Documentation and Training

### Step 1: Create Team Documentation

1. Update your team wiki/confluence with:
   - Branching strategy overview
   - Quick reference guide
   - Common workflows
   - Troubleshooting guide

2. Create video tutorials for:
   - Creating hotfix branches
   - Creating feature branches
   - Promoting releases
   - Handling merge conflicts

### Step 2: Team Training Sessions

Schedule training sessions covering:
- Branching strategy overview (30 min)
- Hands-on workflow practice (1 hour)
- Q&A session (30 min)

## Phase 6: Migration Plan

### Pre-Migration: Service Repository Audit

**Step 1: Inventory All Service Repositories**
```bash
# Create a list of all service repositories
# services.txt
user-service
payment-service
order-service
notification-service
# ... (all 100+ services)
```

**Step 2: Categorize Services**
- **Critical Services**: Services that require extra care during migration
- **Independent Services**: Services with no dependencies (migrate first)
- **Dependent Services**: Services that depend on others (migrate after dependencies)
- **Low-Traffic Services**: Services with minimal changes (can migrate later)

**Step 3: Create Service Dependency Graph**
Document which services depend on which others to determine migration order.

### Week 1: Preparation
- [ ] Review and approve branching strategy
- [ ] Create service repository inventory
- [ ] Identify service dependencies
- [ ] Set up shared workflow templates (if using Option 1)
- [ ] Create migration scripts
- [ ] Set up monitoring and tracking dashboard

### Week 2: Pilot (2-3 Independent Services)
- [ ] Select 2-3 independent services for pilot
- [ ] Apply branching strategy to pilot services:
  - [ ] Create release branches (`release/n+1`, `release/n+2`)
  - [ ] Set up branch protection rules
  - [ ] Configure CI/CD pipelines
  - [ ] Test deployment workflows
- [ ] Monitor and gather feedback
- [ ] Document issues and solutions
- [ ] Adjust strategy if needed

### Week 3-4: Gradual Rollout (25% of Services)
- [ ] Migrate independent services first (no dependencies)
- [ ] Apply to each service repository:
  ```bash
  ./scripts/setup-service-repo.sh <service-name>
  ```
- [ ] Monitor metrics and issues per service
- [ ] Provide support and training to service teams
- [ ] Update documentation based on learnings

### Week 5-6: Expanded Rollout (50% of Services)
- [ ] Migrate dependent services (after their dependencies are migrated)
- [ ] Test cross-service coordination workflows
- [ ] Validate environment promotion flows
- [ ] Continue monitoring and support

### Week 7-8: Full Rollout (Remaining Services)
- [ ] Migrate remaining services
- [ ] Complete documentation updates
- [ ] Archive old branches/processes
- [ ] Conduct retrospective and document lessons learned

### Migration Script Template

Create a script to automate migration for each service:

```bash
#!/bin/bash
# scripts/setup-service-repo.sh

SERVICE_NAME=$1
REPO_URL="https://github.com/your-org/$SERVICE_NAME.git"

if [ -z "$SERVICE_NAME" ]; then
  echo "Usage: $0 <service-name>"
  exit 1
fi

echo "Setting up branching strategy for $SERVICE_NAME..."

# Clone repository
git clone "$REPO_URL" temp-repo
cd temp-repo

# Create release branches
git checkout main
git pull origin main
git checkout -b release/n+1
git push origin release/n+1
git checkout -b release/n+2
git push origin release/n+2
git checkout main

# Copy CI/CD workflow (if using template)
mkdir -p .github/workflows
cp ../templates/deploy.yml .github/workflows/deploy.yml
sed -i "s/SERVICE_NAME/$SERVICE_NAME/g" .github/workflows/deploy.yml

# Commit workflow
git add .github/workflows/deploy.yml
git commit -m "chore: add standardized deployment workflow"
git push origin main

echo "✅ Setup complete for $SERVICE_NAME"
echo "⚠️  Don't forget to:"
echo "   1. Configure branch protection rules in GitHub"
echo "   2. Set up repository secrets and variables"
echo "   3. Test deployment workflow"

cd ..
rm -rf temp-repo
```

### Tracking Migration Progress

Create a tracking spreadsheet or dashboard:

| Service Name | Status | Release Branches | CI/CD Configured | Branch Protection | Notes |
|--------------|--------|------------------|-------------------|-------------------|-------|
| user-service | ✅ Complete | ✅ | ✅ | ✅ | - |
| payment-service | 🟡 In Progress | ✅ | ✅ | ⏳ | Waiting for approval |
| order-service | ⏳ Pending | - | - | - | Scheduled for Week 3 |

## Phase 7: Monitoring and Metrics

### Key Metrics to Track

1. **Deployment Metrics**
   - Time from commit to BLD
   - Time from BLD to PRD
   - Deployment success rate
   - Rollback frequency

2. **Branch Metrics**
   - Average branch lifetime
   - Merge conflict frequency
   - PR review time
   - Hotfix frequency

3. **Release Metrics**
   - Time between releases
   - Feature delivery time
   - Bug rate by release type

### Dashboard Setup

Create dashboards showing:
- Current deployments per environment
- Branch status and PRs
- Deployment pipeline status
- Release timeline

## Phase 8: Continuous Improvement

### Regular Reviews

Schedule monthly reviews to:
- Analyze metrics and identify bottlenecks
- Gather team feedback
- Update strategy based on learnings
- Share best practices

### Iteration

Be prepared to adjust:
- Branch naming conventions
- Environment promotion gates
- CI/CD pipeline configurations
- Documentation and processes

## Troubleshooting Common Issues

### Issue: Developers creating branches from wrong base

**Solution**: 
- Create branch creation helper script
- Add pre-commit hooks to validate branch names
- Provide clear documentation

### Issue: Merge conflicts in release branches

**Solution**:
- Regular sync schedule (daily/weekly)
- Automated sync scripts
- Clear conflict resolution process

### Issue: Unclear which environment a branch deploys to

**Solution**:
- Clear branch naming conventions
- CI/CD pipeline logs showing target environment
- Documentation with visual diagrams

### Issue: Hotfixes not being cherry-picked

**Solution**:
- Automated cherry-pick workflow
- Checklist in PR template
- Monitoring alerts for unmerged hotfixes

## Support and Resources

- **Strategy Document**: `BRANCHING_STRATEGY.md`
- **Quick Reference**: `QUICK_REFERENCE.md`
- **Team Slack Channel**: #branching-strategy
- **Escalation**: [Your DevOps Lead]

## Next Steps

1. Review this implementation guide with your team
2. Customize scripts and configurations for your infrastructure
3. Set up branch protection rules
4. Configure CI/CD pipelines
5. Run pilot with selected services
6. Gather feedback and iterate
7. Roll out to all services

