# Jira Integration Guide

## Overview

All feature and hotfix branches **must** include the Jira ticket ID (CORENGC-xxxx) in the branch name. This ensures traceability between code changes and Jira tickets.

## Branch Naming with Jira IDs

### Required Format

- **Hotfix**: `hotfix/CORENGC-xxxx-description`
- **Feature**: `feature/CORENGC-xxxx-description`
- **Release**: `release/version` (no Jira ID required)

### Examples

✅ **Correct**:
- `hotfix/CORENGC-1234-critical-auth-fix`
- `hotfix/CORENGC-5678-security-patch`
- `feature/CORENGC-1234-user-authentication`
- `feature/CORENGC-5678-payment-integration`

❌ **Incorrect**:
- `hotfix/critical-bug-fix` (missing Jira ID)
- `feature/user-auth` (missing Jira ID)
- `hotfix/CORENGC-1234` (no description, but acceptable)
- `feature/CORENGC-1234` (no description, but acceptable)

## Using the Branch Creation Script

The `create-branch.sh` script validates Jira ID format automatically:

```bash
# Create hotfix branch
./scripts/create-branch.sh hotfix CORENGC-1234 critical-bug-fix

# Create feature branch
./scripts/create-branch.sh feature CORENGC-5678 user-authentication release/n+1

# Create release branch (no Jira ID needed)
./scripts/create-branch.sh release v1.3.0 "" main
```

## Commit Messages

Include Jira ID in commit messages for better traceability:

```bash
# Good commit messages
git commit -m "fix: resolve authentication issue [CORENGC-1234]"
git commit -m "feat: add user authentication [CORENGC-5678]"
git commit -m "fix(CORENGC-1234): resolve critical bug"
```

## Pull Request Titles

Include Jira ID in PR titles:

- ✅ `[CORENGC-1234] Fix critical authentication bug`
- ✅ `[CORENGC-5678] Add user authentication feature`
- ✅ `Fix: Critical bug [CORENGC-1234]`

## CI/CD Integration

The CI/CD pipeline automatically extracts Jira IDs from branch names:

```yaml
# Example: Extract Jira ID from branch name
BRANCH_NAME="${{ github.ref_name }}"
if [[ "$BRANCH_NAME" =~ ^(hotfix|feature)/CORENGC-([0-9]+) ]]; then
  JIRA_ID="CORENGC-${BASH_REMATCH[2]}"
  echo "JIRA_ID=$JIRA_ID" >> $GITHUB_ENV
fi
```

## Jira Integration Benefits

1. **Traceability**: Link code changes to Jira tickets
2. **Automation**: Auto-update Jira tickets on PR merge
3. **Reporting**: Track work by Jira ticket
4. **Compliance**: Audit trail of changes

## GitHub-Jira Integration Setup

### Option 1: GitHub App Integration

1. Install Jira integration app in GitHub
2. Configure to auto-link branches and PRs
3. Set up automatic ticket updates

### Option 2: Webhook Integration

Configure webhooks to:
- Update Jira ticket when PR is created
- Update Jira ticket when PR is merged
- Add deployment status to Jira ticket

### Option 3: Manual Linking

Use Jira smart commits in PR descriptions:
```
[CORENGC-1234] Fix critical bug

Related tickets:
- CORENGC-1234
- CORENGC-5678
```

## Branch Protection Rules

GitHub branch protection can be configured to:
- Require Jira ID in branch names (via regex)
- Block branches without proper Jira ID format
- Validate Jira ticket exists before allowing PR

Example regex for branch protection:
```
^(main|release/.*|hotfix/CORENGC-[0-9]+.*|feature/CORENGC-[0-9]+.*)$
```

## Validation Script

Use this script to validate branch names:

```bash
#!/bin/bash
# scripts/validate-branch-name.sh

BRANCH_NAME=$1

if [[ "$BRANCH_NAME" =~ ^(hotfix|feature)/CORENGC-[0-9]+ ]]; then
  echo "✅ Valid branch name: $BRANCH_NAME"
  JIRA_ID=$(echo "$BRANCH_NAME" | grep -oP 'CORENGC-[0-9]+')
  echo "📋 Jira ticket: $JIRA_ID"
  exit 0
elif [[ "$BRANCH_NAME" =~ ^(main|release/) ]]; then
  echo "✅ Valid branch name: $BRANCH_NAME (no Jira ID required)"
  exit 0
else
  echo "❌ Invalid branch name: $BRANCH_NAME"
  echo "Expected format:"
  echo "  - hotfix/CORENGC-xxxx-description"
  echo "  - feature/CORENGC-xxxx-description"
  echo "  - release/version"
  exit 1
fi
```

## Best Practices

1. **Always include Jira ID**: Even for small fixes, include the Jira ticket ID
2. **Use descriptive names**: Add a brief description after the Jira ID
3. **Keep it short**: Branch names should be concise but clear
4. **Consistent format**: Use kebab-case for descriptions
5. **Link in PRs**: Always link to Jira ticket in PR description

## Troubleshooting

### Issue: Branch name doesn't match expected format

**Solution**: Use the `create-branch.sh` script which validates format automatically.

### Issue: Jira ticket doesn't exist

**Solution**: Create the Jira ticket first, then create the branch with that ticket ID.

### Issue: Multiple Jira tickets for one feature

**Solution**: Use the primary/epic ticket ID in the branch name, mention others in PR description.

## Examples

### Creating a Hotfix

```bash
# Jira ticket: CORENGC-1234
./scripts/create-branch.sh hotfix CORENGC-1234 critical-auth-fix
# Creates: hotfix/CORENGC-1234-critical-auth-fix
```

### Creating a Feature

```bash
# Jira ticket: CORENGC-5678
./scripts/create-branch.sh feature CORENGC-5678 user-authentication release/n+1
# Creates: feature/CORENGC-5678-user-authentication
```

### Cross-Service Feature

```bash
# Same Jira ticket across all services: CORENGC-9012
# In user-service:
./scripts/create-branch.sh feature CORENGC-9012 payment-integration release/n+1

# In payment-service:
./scripts/create-branch.sh feature CORENGC-9012 payment-integration release/n+1

# In api-gateway:
./scripts/create-branch.sh feature CORENGC-9012 payment-integration release/n+1
```

