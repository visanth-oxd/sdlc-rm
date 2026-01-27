# SDLC Release Management - Branching Strategy

This repository contains the branching strategy and implementation guides for managing deployments across multiple environment layers (BLD, INT, PRE, PRD) with different release types (stable, beta, alpha).

**Architecture**: Each service has its own GitHub repository. All service repositories follow the same branching strategy independently, enabling consistent practices across 100+ services while maintaining service autonomy.

## 📚 Documentation

### Strategic Capability
- **[TRUNK_BASED_STRATEGY.md](./TRUNK_BASED_STRATEGY.md)** ⭐ **PRIMARY** - Trunk-based development as strategic capability: continuous iterative delivery

### Core Documentation
- **[BRANCHING_STRATEGY.md](./BRANCHING_STRATEGY.md)** - Comprehensive branching strategy document
- **[TRUNK_VS_RELEASE_BRANCHES.md](./TRUNK_VS_RELEASE_BRANCHES.md)** - Deep analysis: Trunk-based vs Release branches with practical examples
- **[TRUNK_BASED_DEVELOPMENT.md](./TRUNK_BASED_DEVELOPMENT.md)** - Trunk-based development integration guide

### Supporting Documentation
- **[MULTI_REPO_ARCHITECTURE.md](./MULTI_REPO_ARCHITECTURE.md)** - Multi-repository architecture guide (each service = one repo)
- **[JIRA_INTEGRATION.md](./JIRA_INTEGRATION.md)** - Jira ID (CORENGC-xxxx) integration guide
- **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** - Quick reference guide for daily workflows
- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)** - Step-by-step implementation guide
- **[BRANCHING_DIAGRAMS.md](./BRANCHING_DIAGRAMS.md)** - Visual diagrams and workflows

## 🎯 Overview

### Strategic Approach: Trunk-Based Development

**Trunk-based development** is our **primary strategic capability** for continuous iterative code delivery. All features are developed and deployed through the `main` branch using:
- ✅ Short-lived feature branches (hours to 1 day)
- ✅ Small, focused PRs (< 400 lines)
- ✅ Feature flags for gradual rollout
- ✅ Continuous deployment (multiple times per day)
- ✅ Iterative development (break large features into small PRs)

**See [TRUNK_BASED_STRATEGY.md](./TRUNK_BASED_STRATEGY.md)** for the complete strategic guide.

### Environment Model

```
Environment Layers: BLD → INT → PRE → PRD
Release Types: stable (n), beta (n+1), alpha (n+2)
Instances: Multiple per type (stable-01, beta-01, beta-02, etc.)
```

### Branch Structure

- **`main`** ⭐ **PRIMARY** → Production (trunk-based development, continuous delivery)
- **`feature/*`** ⭐ **PRIMARY** → All features merge to main iteratively
- **`hotfix/*`** → Critical production fixes
- **`release/n+1`** → Optional: Beta release (rarely used)
- **`release/n+2`** → Optional: Alpha release (rarely used)

## 🚀 Quick Start

### Creating a Hotfix

```bash
# Jira ticket: CORENGC-1234
git checkout main
git pull origin main
git checkout -b hotfix/CORENGC-1234-critical-bug-fix
# Make changes and commit
git commit -m "fix: resolve critical bug [CORENGC-1234]"
git push origin hotfix/CORENGC-1234-critical-bug-fix
# Create PR: hotfix/CORENGC-1234-critical-bug-fix → main
```

### Creating a Feature Branch (Trunk-Based)

```bash
# Jira ticket: CORENGC-5678
# ALWAYS create from main (trunk-based development)
git checkout main
git pull origin main
git checkout -b feature/CORENGC-5678-new-feature
# Make changes and commit (small, focused changes)
git commit -m "feat: add new feature [CORENGC-5678]"
git push origin feature/CORENGC-5678-new-feature
# Create PR: feature/CORENGC-5678-new-feature → main
# Merge same day → automatic deployment to production
```

## 📊 Branch-to-Environment Mapping

| Branch | Environments | Version |
|--------|--------------|---------|
| `main` | BLD-stable-01 → INT-stable-01 → PRE-stable-01 → PRD-stable-01 | n |
| `release/n+1` | BLD-beta-* → INT-beta-* → PRE-beta-* | n+1 |
| `release/n+2` | BLD-alpha-* → INT-alpha-* | n+2 |
| `hotfix/*` | BLD-stable-01 → INT-stable-01 → PRE-stable-01 → PRD-stable-01 | n |

## 🔄 Workflow Diagrams

### Hotfix Flow
```
main → hotfix/bug-fix → BLD-stable-01 → INT-stable-01 → PRE-stable-01 → PRD-stable-01
         ↓
    (merge back to main)
         ↓
    (cherry-pick to release branches)
```

### Feature Development Flow
```
release/n+1 → feature/new-feature → BLD-beta-01 → INT-beta-01 → PRE-beta-01
         ↑
    (merge back)
```

### Release Promotion Flow
```
release/n+1 (beta) → main (stable) → Production
release/n+2 (alpha) → release/n+1 (beta)
```

## 📋 Key Principles

1. **`main` is production**: Always reflects what's deployed to PRD-stable-01
2. **Trunk-based development**: All features merge to `main` iteratively (primary strategic capability)
3. **Short-lived branches**: Feature branches live for hours to 1 day maximum
4. **Small, focused PRs**: Each PR < 400 lines (ideally < 200 lines)
5. **Feature flags**: All new features deployed behind flags for gradual rollout
6. **Continuous deployment**: Deploy multiple times per day
7. **Iterative development**: Break large features into small, deployable increments
8. **Hotfixes start from `main`**: Critical fixes flow through stable environments
9. **Environment promotion**: BLD → INT → PRE → PRD (with appropriate gates)
10. **Service independence**: Each service has its own GitHub repository, following the same strategy
11. **Scalable**: Works for 100s of services, each with independent release cycles
12. **Jira integration**: All feature and hotfix branches must include Jira ID (CORENGC-xxxx)

## 🛠️ Implementation

See [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md) for:
- Repository setup
- CI/CD pipeline configuration
- Branch protection rules
- Automation scripts
- Migration plan

## 📖 Best Practices

- Use conventional commits: `feat:`, `fix:`, `docs:`, etc.
- Keep branches focused and small
- Delete branches after merge
- Tag releases appropriately
- Regular sync with base branches

## 🔍 Common Scenarios

### Scenario 1: Critical Production Bug
1. Create `hotfix/CORENGC-1234-critical-fix` from `main` (Jira: CORENGC-1234)
2. Fix and test in BLD-stable-01
3. Deploy through INT → PRE → PRD
4. Merge back to `main`
5. Cherry-pick to release branches

### Scenario 2: New Feature (Trunk-Based)
1. Create `feature/CORENGC-5678-new-feature` from `main` (Jira: CORENGC-5678)
2. Develop iteratively (break into small PRs if large feature)
3. Create PR → `main` (same day)
4. Merge and deploy automatically
5. Use feature flags for gradual rollout (10% → 50% → 100%)
6. **Result**: Feature in production same day or next day

### Scenario 3: Release Promotion
1. When `release/n+1` is ready, merge to `main`
2. Tag `main` with new version
3. Deploy through stable environments
4. Create new `release/n+2` from `main`

## 📞 Support

- **Questions**: See [BRANCHING_STRATEGY.md](./BRANCHING_STRATEGY.md)
- **Quick Help**: See [QUICK_REFERENCE.md](./QUICK_REFERENCE.md)
- **Implementation**: See [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)

## 🔄 Version History

- **v1.0.0** - Initial branching strategy documentation

## 📝 License

Internal documentation - For team use only

