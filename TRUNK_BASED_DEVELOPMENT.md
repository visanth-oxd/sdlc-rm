# Trunk-Based Development Integration

## Overview

This document describes how to integrate **trunk-based development** into the existing branching strategy to enable faster delivery to production while maintaining release branches for longer-term features.

## What is Trunk-Based Development?

Trunk-based development is a practice where:
- Developers work on **short-lived feature branches** (hours to days, not weeks)
- Features merge **directly to `main`** (the trunk) frequently
- **Continuous integration** ensures `main` is always deployable
- **Feature flags** control feature rollout in production
- Faster feedback loops and quicker time-to-production

## Hybrid Approach: Trunk + Release Branches

Our strategy supports **both approaches**:

### Fast Track: Direct to Production (Trunk-Based)
- **For**: Small features, bug fixes, improvements ready for immediate production
- **Flow**: `main` → `feature/CORENGC-xxxx` → `main` → Production
- **Timeline**: Hours to days
- **Use when**: Feature is production-ready, low risk, doesn't need extended testing

### Release Track: Through Release Branches
- **For**: Large features, experimental work, features needing extended testing
- **Flow**: `release/n+1` → `feature/CORENGC-xxxx` → `release/n+1` → Beta → `main` → Production
- **Timeline**: Weeks to months
- **Use when**: Feature needs beta/alpha testing, is part of a planned release

## Decision Matrix: Which Path to Use?

```
Is the feature small and production-ready?
├─ YES → Fast Track (Trunk-Based)
│   └─ Merge directly to main
│
└─ NO → Release Track
    ├─ Is it experimental/future work?
    │   └─ YES → Use release/n+2 (alpha)
    │
    └─ Is it part of next release?
        └─ YES → Use release/n+1 (beta)
```

### Fast Track Criteria
✅ **Use trunk-based (direct to main) when**:
- Small, focused changes (< 200 lines typically)
- Low risk, well-tested locally
- Can be deployed immediately
- Doesn't require coordination with other services
- Can be feature-flagged if needed

### Release Track Criteria
✅ **Use release branches when**:
- Large features requiring multiple PRs
- Needs extended testing in beta/alpha environments
- Part of a coordinated release across services
- Experimental or breaking changes
- Requires stakeholder approval before production

## Trunk-Based Development Workflow

### 1. Create Short-Lived Feature Branch

```bash
# Jira ticket: CORENGC-1234
# Create from main (not release branch)
git checkout main
git pull origin main
git checkout -b feature/CORENGC-1234-small-improvement

# Or use the helper script
./scripts/create-branch.sh feature CORENGC-1234 small-improvement main
```

**Key Points**:
- Branch from `main` (the trunk)
- Keep branch lifetime **< 1 day** ideally
- Small, focused changes
- One logical unit of work

### 2. Develop and Test Locally

```bash
# Make changes
# Run tests locally
npm test
npm run lint

# Commit frequently with clear messages
git commit -m "feat: add user preference toggle [CORENGC-1234]"
```

### 3. Create PR to Main

```bash
git push origin feature/CORENGC-1234-small-improvement
# Create PR: feature/CORENGC-1234-small-improvement → main
```

**PR Requirements**:
- ✅ All CI/CD checks pass
- ✅ Code review approved (1-2 reviewers)
- ✅ Tests added/updated
- ✅ No merge conflicts
- ✅ Small, focused change

### 4. Merge to Main (Trunk)

```bash
# After PR approval, merge to main
# This triggers automatic deployment to BLD-stable-01
```

**What Happens**:
1. PR merges to `main`
2. CI/CD automatically deploys to `BLD-stable-01`
3. Automated tests run
4. If successful, promotes to `INT-stable-01`
5. Can promote to `PRE-stable-01` → `PRD-stable-01` when ready

### 5. Feature Flags (Optional)

For features that need gradual rollout:

```typescript
// Use feature flags to control rollout
if (featureFlags.isEnabled('new-payment-method')) {
  // New feature code
} else {
  // Existing code
}
```

**Benefits**:
- Deploy to production but keep feature hidden
- Gradual rollout (10% → 50% → 100%)
- Quick rollback if issues arise
- A/B testing capabilities

## Release Branch Workflow (Existing)

For larger features, continue using release branches:

```bash
# Large feature for next release
git checkout release/n+1
git checkout -b feature/CORENGC-5678-large-feature
# Develop over days/weeks
# Merge to release/n+1 when ready
# Test in beta environments
# Eventually merge release/n+1 → main
```

## Key Principles

### 1. Keep Main Always Deployable
- ✅ `main` should always be in a deployable state
- ✅ All tests pass on `main`
- ✅ No broken builds
- ✅ Quick fixes if `main` breaks

### 2. Short-Lived Branches
- ✅ Feature branches live for **hours to 1-2 days max**
- ✅ If branch lives longer, consider splitting into smaller PRs
- ✅ Delete branches immediately after merge

### 3. Small, Focused Changes
- ✅ One logical change per PR
- ✅ Easier to review
- ✅ Faster to merge
- ✅ Lower risk

### 4. Continuous Integration
- ✅ Every commit to `main` triggers CI/CD
- ✅ Fast feedback (< 10 minutes)
- ✅ Automatic deployment to BLD
- ✅ Fail fast, fix fast

### 5. Feature Flags
- ✅ Use feature flags for gradual rollout
- ✅ Deploy code without exposing feature
- ✅ Enable/disable without redeployment
- ✅ Quick rollback capability

## Branch Strategy Summary

| Branch Type | Source | Target | Lifetime | Use Case |
|-------------|--------|--------|----------|----------|
| `feature/*` (trunk) | `main` | `main` | Hours-1 day | Small, production-ready features |
| `feature/*` (release) | `release/n+1` or `n+2` | `release/n+1` or `n+2` | Days-weeks | Large features, experimental work |
| `hotfix/*` | `main` | `main` | Hours | Critical production fixes |
| `release/n+1` | `main` | `main` (eventually) | Weeks-months | Beta release features |
| `release/n+2` | `main` | `release/n+1` (eventually) | Weeks-months | Alpha/experimental features |

## CI/CD Pipeline Updates

### Fast Track Pipeline (Trunk-Based)

```yaml
# On merge to main
- Trigger: Push to main
- Steps:
  1. Run full test suite (< 5 minutes)
  2. Build application
  3. Deploy to BLD-stable-01
  4. Run smoke tests
  5. Auto-promote to INT-stable-01 if tests pass
  6. Manual approval for PRE → PRD
```

### Release Track Pipeline (Existing)

```yaml
# On merge to release/n+1
- Trigger: Push to release/n+1
- Steps:
  1. Run full test suite
  2. Deploy to BLD-beta-01
  3. Run integration tests
  4. Manual promotion through beta environments
```

## Feature Flag Strategy

### Implementation

```typescript
// Feature flag service
class FeatureFlags {
  isEnabled(feature: string): boolean {
    // Check environment variable, database, or feature flag service
    return process.env[`FEATURE_${feature}`] === 'true';
  }
  
  getRolloutPercentage(feature: string): number {
    // Get rollout percentage from config
    return parseInt(process.env[`FEATURE_${feature}_PERCENTAGE}`] || '0');
  }
}
```

### Rollout Process

1. **Deploy with flag OFF** (0% rollout)
   - Code is in production but disabled
   - Monitor for any issues

2. **Gradual Rollout** (10% → 50% → 100%)
   - Enable for small percentage of users
   - Monitor metrics and errors
   - Increase gradually if stable

3. **Full Rollout** (100%)
   - Enable for all users
   - Continue monitoring

4. **Cleanup** (after stable)
   - Remove feature flag code
   - Simplify codebase

## Best Practices

### 1. Branch Size
- ✅ Keep PRs small (< 400 lines changed)
- ✅ One logical change per PR
- ✅ Split large features into multiple PRs

### 2. Review Speed
- ✅ Request reviews immediately
- ✅ Respond to review comments quickly
- ✅ Use draft PRs for work-in-progress

### 3. Testing
- ✅ Write tests before or with code
- ✅ Run tests locally before pushing
- ✅ Ensure CI/CD passes before requesting review

### 4. Communication
- ✅ Update PR description with context
- ✅ Link to Jira ticket
- ✅ Mention if feature-flagged
- ✅ Notify team of breaking changes

### 5. Monitoring
- ✅ Monitor deployments closely
- ✅ Set up alerts for errors
- ✅ Track feature flag metrics
- ✅ Quick rollback if needed

## Migration to Trunk-Based Development

### Phase 1: Enable Direct Merges to Main
1. Update branch protection rules to allow feature branches to merge to main
2. Update CI/CD to handle feature branches from main
3. Train team on trunk-based principles

### Phase 2: Implement Feature Flags
1. Set up feature flag service (LaunchDarkly, Flagsmith, or custom)
2. Add feature flag infrastructure to services
3. Document feature flag usage

### Phase 3: Optimize CI/CD
1. Speed up test suites (< 10 minutes)
2. Enable parallel test execution
3. Implement deployment automation
4. Set up monitoring and alerts

### Phase 4: Team Adoption
1. Start with small, low-risk changes
2. Gradually increase trunk-based development
3. Keep release branches for large features
4. Monitor and adjust based on results

## Example: Trunk-Based Feature

### Scenario: Small UI Improvement

```bash
# 1. Create branch from main
git checkout main
git pull origin main
git checkout -b feature/CORENGC-1234-button-styling

# 2. Make small change (30 minutes of work)
# Update button styling

# 3. Test locally
npm test
npm run lint

# 4. Commit and push
git commit -m "feat: improve button styling [CORENGC-1234]"
git push origin feature/CORENGC-1234-button-styling

# 5. Create PR to main
# PR: feature/CORENGC-1234-button-styling → main

# 6. Get review (within hours)
# Merge after approval

# 7. Automatic deployment
# CI/CD deploys to BLD-stable-01 → INT-stable-01
# Promote to PRE → PRD when ready

# Total time: Same day deployment to production
```

## Example: Release Branch Feature

### Scenario: Large Payment Feature

```bash
# 1. Create branch from release/n+1
git checkout release/n+1
git checkout -b feature/CORENGC-5678-new-payment-method

# 2. Develop over 2 weeks
# Multiple commits, multiple PRs

# 3. Merge to release/n+1
# Test in beta environments

# 4. Eventually merge release/n+1 → main
# Deploy to production

# Total time: 2-4 weeks to production
```

## Monitoring and Metrics

### Key Metrics to Track

1. **Time to Production**
   - Trunk-based: Target < 1 day
   - Release-based: Target 2-4 weeks

2. **Branch Lifetime**
   - Trunk-based: Target < 1 day
   - Release-based: Days to weeks

3. **PR Size**
   - Target: < 400 lines changed
   - Monitor average PR size

4. **Deployment Frequency**
   - Target: Multiple deployments per day
   - Track deployments to production

5. **Rollback Rate**
   - Monitor how often we need to rollback
   - Target: < 1% of deployments

## Troubleshooting

### Issue: Main branch breaks frequently

**Solutions**:
- Improve test coverage
- Run more tests in CI/CD
- Smaller PRs
- Better code reviews
- Feature flags for risky changes

### Issue: PRs take too long to review

**Solutions**:
- Smaller PRs (easier to review)
- Clear PR descriptions
- Request reviews early
- Set review SLAs
- Use draft PRs for WIP

### Issue: Conflicts with release branches

**Solutions**:
- Regularly sync release branches with main
- Cherry-pick trunk changes to release branches
- Automated sync scripts
- Clear communication about changes

## Conclusion

Trunk-based development enables:
- ✅ **Faster delivery**: Hours to days instead of weeks
- ✅ **Faster feedback**: Quick iteration cycles
- ✅ **Lower risk**: Small, frequent changes
- ✅ **Better quality**: Continuous integration
- ✅ **Flexibility**: Feature flags for gradual rollout

Release branches remain valuable for:
- ✅ Large, coordinated releases
- ✅ Extended testing periods
- ✅ Experimental features
- ✅ Planned release cycles

The hybrid approach gives teams the flexibility to choose the right path for each feature.


