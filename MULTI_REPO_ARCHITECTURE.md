# Multi-Repository Architecture Guide

## Overview

This document clarifies how the branching strategy applies to a multi-repository architecture where **each service has its own GitHub repository**.

## Architecture Model

```
Organization: your-org/
├── user-service/          (GitHub repository)
│   ├── main
│   ├── release/n+1
│   ├── release/n+2
│   └── feature/*, hotfix/*
│
├── payment-service/       (GitHub repository)
│   ├── main
│   ├── release/n+1
│   ├── release/n+2
│   └── feature/*, hotfix/*
│
├── order-service/         (GitHub repository)
│   ├── main
│   ├── release/n+1
│   ├── release/n+2
│   └── feature/*, hotfix/*
│
└── ... (100+ service repositories)
```

## Key Principles

### 1. Service Independence
- ✅ Each service repository operates independently
- ✅ Services can have different release cycles
- ✅ Services can deploy independently to their environments
- ✅ Hotfixes are service-specific (only affect the service that needs fixing)
- ✅ Features are service-specific (developed within the service repository)

### 2. Consistent Strategy
- ✅ All service repositories follow the same branching strategy
- ✅ Same branch names: `main`, `release/n+1`, `release/n+2`, `hotfix/*`, `feature/*`
- ✅ Same environment mapping: `main` → stable, `release/n+1` → beta, `release/n+2` → alpha
- ✅ Same CI/CD workflow structure (customized per service)

### 3. Cross-Service Coordination
When multiple services need to work together:
- Use consistent feature branch naming: `feature/epic-name` across repositories
- Coordinate PR reviews and merges
- Deploy services in dependency order
- Use release coordination tools/scripts

## Repository Setup Per Service

Each service repository should have:

### Branch Structure
```
main                    → Production (stable environments)
release/n+1            → Beta release (beta environments)
release/n+2            → Alpha release (alpha environments)
hotfix/*               → Critical production fixes
feature/*              → New feature development
```

### CI/CD Configuration
- `.github/workflows/deploy.yml` - Standardized deployment workflow
- Service-specific environment variables and secrets
- Service-specific deployment targets

### Branch Protection
- `main`: 2 approvals required, all CI/CD checks must pass
- `release/*`: 1 approval required, relevant CI/CD checks must pass
- `hotfix/*`: 2 approvals required, all CI/CD checks must pass

## Setting Up a Service Repository

### Quick Setup

Use the provided script to set up a new service repository:

```bash
./scripts/setup-service-repo.sh <service-name>
```

This script will:
1. Clone the service repository
2. Create `release/n+1` and `release/n+2` branches
3. Copy CI/CD workflow template
4. Configure service-specific settings

### Manual Setup

1. **Create Release Branches**:
   ```bash
   git checkout main
   git checkout -b release/n+1
   git push origin release/n+1
   git checkout -b release/n+2
   git push origin release/n+2
   ```

2. **Set Up CI/CD**:
   - Copy `.github/workflows/deploy.yml.example` to `.github/workflows/deploy.yml`
   - Customize service name and environment variables

3. **Configure Branch Protection**:
   - GitHub: Settings → Branches → Add protection rules
   - Protect `main` and `release/*` branches

4. **Set Up Secrets**:
   - GitHub: Settings → Secrets and variables → Actions
   - Add service-specific secrets (deployment keys, API keys, etc.)

## Cross-Service Workflows

### Coordinated Feature Development

When a feature spans multiple services:

1. **Create Feature Branches** (same name with Jira ID across services):
   ```bash
   # Jira ticket: CORENGC-5678
   # In user-service repository
   git checkout -b feature/CORENGC-5678-new-payment-method release/n+1
   
   # In payment-service repository
   git checkout -b feature/CORENGC-5678-new-payment-method release/n+1
   
   # In api-gateway repository
   git checkout -b feature/CORENGC-5678-new-payment-method release/n+1
   ```

2. **Develop Independently**:
   - Each service team works in their own repository
   - Coordinate through team channels/meetings

3. **Coordinate PRs**:
   - Create PRs in all affected services
   - Use same reviewers across services
   - Merge simultaneously or in dependency order

4. **Deploy in Order**:
   - Deploy dependencies first (e.g., `user-service`)
   - Then dependent services (e.g., `payment-service`)
   - Finally, gateway/API services (e.g., `api-gateway`)

### Coordinated Release

Use the coordination script:

```bash
./scripts/coordinated-release.sh release/n+1 user-service payment-service order-service
```

This ensures all services are on the same release branch before deployment.

### Service Dependencies

Maintain a service dependency graph:

```
user-service (no dependencies)
  └── payment-service (depends on user-service)
      └── order-service (depends on payment-service, user-service)
          └── api-gateway (depends on all services)
```

**Deployment Order**:
1. `user-service` → Deploy first
2. `payment-service` → Deploy after user-service
3. `order-service` → Deploy after payment-service and user-service
4. `api-gateway` → Deploy last

## Environment Mapping Per Service

Each service deploys to its own set of environments:

### Service: `user-service`
- `main` → `user-service-BLD-stable-01` → `user-service-INT-stable-01` → `user-service-PRE-stable-01` → `user-service-PRD-stable-01`
- `release/n+1` → `user-service-BLD-beta-01` → `user-service-INT-beta-01` → `user-service-PRE-beta-01`
- `release/n+2` → `user-service-BLD-alpha-01` → `user-service-INT-alpha-01`

### Service: `payment-service`
- `main` → `payment-service-BLD-stable-01` → `payment-service-INT-stable-01` → `payment-service-PRE-stable-01` → `payment-service-PRD-stable-01`
- `release/n+1` → `payment-service-BLD-beta-01` → `payment-service-INT-beta-01` → `payment-service-PRE-beta-01`
- `release/n+2` → `payment-service-BLD-alpha-01` → `payment-service-INT-alpha-01`

**Note**: Environment naming can be customized per your infrastructure. The key is consistency in mapping branches to environments.

## CI/CD Workflow Per Service

Each service repository should have a standardized CI/CD workflow that:

1. **Detects Branch Type**: Determines if branch is `main`, `release/n+1`, `release/n+2`, `hotfix/*`, or `feature/*`
2. **Maps to Environment**: Determines target environment based on branch type
3. **Builds and Tests**: Runs build, tests, security scans
4. **Deploys**: Deploys to appropriate BLD environment
5. **Promotes**: Supports promotion through INT → PRE → PRD (for stable)

See `.github/workflows/deploy.yml.example` for a complete template.

## Service-Specific Configuration

### Environment Variables

Each service repository should have:

**GitHub Secrets**:
- `SERVICE_NAME`: Name of the service (e.g., `user-service`)
- `DEPLOYMENT_KEY`: Service-specific deployment key
- Service-specific API keys, database credentials, etc.

**GitHub Variables**:
- `ENVIRONMENT_URLS`: JSON mapping of environments to URLs
- `DEPLOYMENT_CONFIG`: Service-specific deployment configuration

### Deployment Configuration

Each service may have different:
- Build commands
- Test suites
- Deployment targets
- Environment-specific configurations

These should be configured in the service repository's CI/CD workflow.

## Monitoring Across Services

### Service-Level Metrics

Track per service:
- Deployment frequency
- Deployment success rate
- Time to production
- Hotfix frequency
- Feature delivery time

### Organization-Level Metrics

Aggregate across all services:
- Total deployments per day/week
- Cross-service release coordination success rate
- Service dependency impact on releases
- Overall system stability

## Best Practices

### 1. Service Autonomy
- ✅ Allow services to release independently
- ✅ Don't force all services to release together
- ✅ Coordinate only when necessary (cross-service features)

### 2. Consistency
- ✅ Use same branch names across all services
- ✅ Use same CI/CD workflow structure
- ✅ Follow same code review and merge processes

### 3. Communication
- ✅ Use team channels for cross-service coordination
- ✅ Document service dependencies
- ✅ Share release plans across teams

### 4. Tooling
- ✅ Use shared CI/CD workflow templates
- ✅ Automate service setup with scripts
- ✅ Use release coordination tools for complex releases

## Migration Strategy

When migrating existing services:

1. **Start with Independent Services**: Migrate services with no dependencies first
2. **Use Setup Script**: Use `./scripts/setup-service-repo.sh` for consistency
3. **Test Thoroughly**: Verify deployments work before migrating dependent services
4. **Coordinate Dependencies**: Migrate dependent services after their dependencies are migrated
5. **Monitor Closely**: Track metrics and issues during migration

See [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md) for detailed migration plan.

## Troubleshooting

### Issue: Service A depends on Service B, but B hasn't deployed yet

**Solution**: 
- Deploy services in dependency order
- Use health checks to verify dependencies are ready
- Implement service discovery and graceful degradation

### Issue: Multiple services need to release together

**Solution**:
- Use `coordinated-release.sh` script
- Create a release plan document
- Coordinate through team channels
- Use deployment orchestration tools

### Issue: Inconsistent branch names across services

**Solution**:
- Standardize on branch naming convention
- Use setup script to ensure consistency
- Document naming conventions clearly
- Regular audits of repository configurations

## Resources

- **[BRANCHING_STRATEGY.md](./BRANCHING_STRATEGY.md)** - Complete branching strategy
- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)** - Step-by-step implementation
- **[QUICK_REFERENCE.md](./QUICK_REFERENCE.md)** - Quick reference guide
- **Scripts**: `./scripts/setup-service-repo.sh`, `./scripts/coordinated-release.sh`

