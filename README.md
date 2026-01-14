# SDLC Release Management - Branching Strategy

This repository contains the branching strategy and implementation guides for managing deployments across multiple environment layers (BLD, INT, PRE, PRD) with different release types (stable, beta, alpha).

**Architecture**: Each service has its own GitHub repository. All service repositories follow the same branching strategy independently, enabling consistent practices across 100+ services while maintaining service autonomy.

## 📚 Documentation

- **[BRANCHING_STRATEGY.md](./BRANCHING_STRATEGY.md)** - Comprehensive branching strategy document
- **[MULTI_REPO_ARCHITECTURE.md](./MULTI_REPO_ARCHITECTURE.md)** - Multi-repository architecture guide (each service = one repo)
- **[JIRA_INTEGRATION.md](./JIRA_INTEGRATION.md)** - Jira ID (CORENGC-xxxx) integration guide
- **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** - Quick reference guide for daily workflows
- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)** - Step-by-step implementation guide
- **[BRANCHING_DIAGRAMS.md](./BRANCHING_DIAGRAMS.md)** - Visual diagrams and workflows

## 🎯 Overview

### Environment Model

```
Environment Layers: BLD → INT → PRE → PRD
Release Types: stable (n), beta (n+1), alpha (n+2)
Instances: Multiple per type (stable-01, beta-01, beta-02, etc.)
```

### Branch Structure

- **`main`** → Production (stable environments)
- **`release/n+1`** → Beta release (beta environments)
- **`release/n+2`** → Alpha release (alpha environments)
- **`hotfix/*`** → Critical production fixes
- **`feature/*`** → New feature development

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

### Creating a Feature Branch

```bash
# Jira ticket: CORENGC-5678
git checkout release/n+1
git pull origin release/n+1
git checkout -b feature/CORENGC-5678-new-feature
# Make changes and commit
git commit -m "feat: add new feature [CORENGC-5678]"
git push origin feature/CORENGC-5678-new-feature
# Create PR: feature/CORENGC-5678-new-feature → release/n+1
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
2. **Hotfixes start from `main`**: Critical fixes flow through stable environments
3. **Features target release branches**: Beta features go to `release/n+1`, alpha to `release/n+2`
4. **Environment promotion**: BLD → INT → PRE → PRD (with appropriate gates)
5. **Service independence**: Each service has its own GitHub repository, following the same strategy
6. **Scalable**: Works for 100s of services, each with independent release cycles
7. **Jira integration**: All feature and hotfix branches must include Jira ID (CORENGC-xxxx)

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

### Scenario 2: New Feature for Next Release
1. Create `feature/CORENGC-5678-new-feature` from `release/n+1` (Jira: CORENGC-5678)
2. Develop and test in beta environments
3. Merge to `release/n+1`
4. Promote through beta environments

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

