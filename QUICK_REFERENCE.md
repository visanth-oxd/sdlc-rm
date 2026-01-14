# Branching Strategy Quick Reference

## Branch-to-Environment Mapping

| Branch | Deploys To | Version | Purpose |
|--------|------------|---------|---------|
| `main` | BLD-stable-01 → INT-stable-01 → PRE-stable-01 → PRD-stable-01 | n (Production) | Production-ready code |
| `release/n+1` | BLD-beta-* → INT-beta-* → PRE-beta-* | n+1 (Beta) | Next release candidate |
| `release/n+2` | BLD-alpha-* → INT-alpha-* | n+2 (Alpha) | Future release candidate |
| `hotfix/*` | BLD-stable-01 → INT-stable-01 → PRE-stable-01 → PRD-stable-01 | n (Production) | Critical production fixes |
| `feature/*` | Based on source branch (beta/alpha) | n+1 or n+2 | New feature development |

## Common Workflows

### 🔥 Hotfix (Production Bug)
```bash
# Jira ticket: CORENGC-1234
git checkout main
git pull origin main
git checkout -b hotfix/CORENGC-1234-critical-bug-fix
# Make changes
git commit -m "fix: resolve critical bug [CORENGC-1234]"
git push origin hotfix/CORENGC-1234-critical-bug-fix
# Create PR: hotfix/CORENGC-1234-critical-bug-fix → main
# After merge, cherry-pick to release branches if needed
```

### ✨ New Feature (Beta Release)
```bash
# Jira ticket: CORENGC-5678
git checkout release/n+1
git pull origin release/n+1
git checkout -b feature/CORENGC-5678-new-feature
# Make changes
git commit -m "feat: add new feature [CORENGC-5678]"
git push origin feature/CORENGC-5678-new-feature
# Create PR: feature/CORENGC-5678-new-feature → release/n+1
```

### 🚀 New Feature (Alpha Release)
```bash
# Jira ticket: CORENGC-9012
git checkout release/n+2
git pull origin release/n+2
git checkout -b feature/CORENGC-9012-future-feature
# Make changes
git commit -m "feat: add experimental feature [CORENGC-9012]"
git push origin feature/CORENGC-9012-future-feature
# Create PR: feature/CORENGC-9012-future-feature → release/n+2
```

### 📦 Release Promotion (Beta → Stable)
```bash
# When release/n+1 is ready for production
# Create PR: release/n+1 → main
# After merge:
git checkout main
git pull origin main
git tag v1.3.0
git push origin v1.3.0
# Create new release branch:
git checkout -b release/n+2
git push origin release/n+2
```

## Branch Naming Conventions

- **Hotfix**: `hotfix/CORENGC-xxxx-description` (Jira ID required)
  - Example: `hotfix/CORENGC-1234-security-patch`, `hotfix/CORENGC-5678-auth-fix`
  - Format: `hotfix/CORENGC-<ticket-number>-<description>`
  
- **Feature**: `feature/CORENGC-xxxx-description` (Jira ID required)
  - Example: `feature/CORENGC-1234-user-authentication`, `feature/CORENGC-5678-payment-integration`
  - Format: `feature/CORENGC-<ticket-number>-<description>`
  
- **Release**: `release/version` or `release/quarter`
  - Example: `release/v1.3.0`, `release/2024-Q2`
  - Note: Release branches don't include Jira IDs

## Environment Promotion Flow

```
BLD → INT → PRE → PRD
 ↑      ↑     ↑     ↑
 │      │     │     │
Auto   Auto  Manual Manual
```

- **BLD → INT**: Automatic on successful tests
- **INT → PRE**: Automatic on successful integration tests
- **PRE → PRD**: Manual approval required

## Commit Message Format

Use [Conventional Commits](https://www.conventionalcommits.org/):

- `feat: add new feature` - New feature
- `fix: resolve bug` - Bug fix
- `docs: update documentation` - Documentation changes
- `chore: update dependencies` - Maintenance tasks
- `refactor: restructure code` - Code refactoring
- `test: add unit tests` - Test additions
- `perf: improve performance` - Performance improvements

Include ticket reference: `fix: resolve authentication issue [CORENGC-1234]`

## Pull Request Checklist

- [ ] Branch is up to date with target branch
- [ ] All CI/CD checks pass
- [ ] Code follows style guidelines
- [ ] Tests added/updated
- [ ] Documentation updated (if needed)
- [ ] Commit messages follow convention
- [ ] Appropriate reviewers assigned
- [ ] Description includes context and testing instructions

## Branch Protection Summary

| Branch Type | Reviews Required | Status Checks | Direct Push |
|-------------|------------------|---------------|-------------|
| `main` | 2 approvals | All pipelines | ❌ Blocked |
| `release/*` | 1 approval | Relevant pipelines | ❌ Blocked |
| `hotfix/*` | 2 approvals | All pipelines | ❌ Blocked |
| `feature/*` | 1 approval | Unit tests | ✅ Allowed |

## Common Commands

### Create and Switch Branches
```bash
# Hotfix from main (Jira ticket: CORENGC-1234)
git checkout -b hotfix/CORENGC-1234-bug-fix main

# Feature from release branch (Jira ticket: CORENGC-5678)
git checkout -b feature/CORENGC-5678-new-feature release/n+1
```

### Sync with Upstream
```bash
# Update main
git checkout main
git pull origin main

# Update release branch
git checkout release/n+1
git pull origin release/n+1
```

### Cherry-pick Hotfix to Release Branches
```bash
# After merging hotfix to main
git checkout release/n+1
git cherry-pick <commit-hash>
git push origin release/n+1

git checkout release/n+2
git cherry-pick <commit-hash>
git push origin release/n+2
```

### Clean Up Merged Branches
```bash
# Delete local branches that have been merged
git branch --merged | grep -v "\*\|main\|release" | xargs -n 1 git branch -d

# Delete remote branches (after PR merge)
git push origin --delete branch-name
```

## Version Tagging

```bash
# Tag stable release
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3

# Tag beta release
git tag -a v1.3.0-beta.1 -m "Beta release v1.3.0-beta.1"
git push origin v1.3.0-beta.1

# Tag alpha release
git tag -a v1.4.0-alpha.1 -m "Alpha release v1.4.0-alpha.1"
git push origin v1.4.0-alpha.1
```

## Troubleshooting

### Merge Conflicts
1. Update your branch: `git pull origin target-branch`
2. Resolve conflicts locally
3. Test thoroughly
4. Push and update PR

### Wrong Base Branch
1. Close current PR
2. Create new branch from correct base
3. Cherry-pick commits: `git cherry-pick <commit-range>`
4. Create new PR

### Need to Update Release Branch
1. Checkout release branch: `git checkout release/n+1`
2. Merge from main: `git merge main` (or cherry-pick specific commits)
3. Resolve conflicts if any
4. Push: `git push origin release/n+1`

## Support Contacts

- **Branch Strategy Questions**: [Your Team Contact]
- **CI/CD Issues**: [DevOps Team Contact]
- **Environment Access**: [Infrastructure Team Contact]

