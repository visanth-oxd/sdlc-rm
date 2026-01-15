# Release Orchestration Integration Guide

## Overview

This document explains how the branching strategy integrates with release orchestration tools like **Harness** to enable automated deployments, environment management, and coordinated releases across multiple services.

## How Branching Strategy Enables Release Orchestration

### 1. Clear Branch-to-Environment Mapping

The branching strategy provides a **deterministic mapping** between branches and environments:

| Branch Pattern | Target Environment | Version |
|----------------|-------------------|---------|
| `main` | `BLD-stable-01` → `INT-stable-01` → `PRE-stable-01` → `PRD-stable-01` | n (Production) |
| `release/n+1` | `BLD-beta-01` → `INT-beta-01` → `PRE-beta-01` | n+1 (Beta) |
| `release/n+2` | `BLD-alpha-01` → `INT-alpha-01` | n+2 (Alpha) |
| `hotfix/CORENGC-*` | `BLD-stable-01` → `INT-stable-01` → `PRE-stable-01` → `PRD-stable-01` | n (Production) |
| `feature/CORENGC-*` (from main) | `BLD-stable-01` → `INT-stable-01` | n (Trunk-based) |
| `feature/CORENGC-*` (from release/n+1) | `BLD-beta-01` → `INT-beta-01` | n+1 (Beta) |

This mapping enables **automated environment selection** in orchestration tools.

### 2. Environment Creation Strategy

New environments can be created based on branch patterns:

#### Pattern-Based Environment Creation

```
Branch Pattern: release/n+1
  → Creates: BLD-beta-02, INT-beta-02, PRE-beta-02 (if needed)

Branch Pattern: release/n+2
  → Creates: BLD-alpha-02, INT-alpha-02 (if needed)

Branch Pattern: feature/CORENGC-*
  → Creates: BLD-feature-CORENGC-* (temporary, auto-cleanup)
```

#### Environment Naming Convention

```
{LAYER}-{RELEASE_TYPE}-{INSTANCE}

Examples:
- BLD-stable-01
- INT-beta-02
- PRE-alpha-01
- PRD-stable-01
```

## Harness Release Orchestration Integration

### Harness Pipeline Configuration

#### 1. Branch-Based Pipeline Triggers

```yaml
# Harness Pipeline: service-deployment.yml
pipeline:
  name: Service Deployment Pipeline
  identifier: service_deployment
  
  # Trigger on branch push
  triggers:
    - name: Branch Push Trigger
      identifier: branch_push
      type: Webhook
      spec:
        type: Github
        spec:
          connectorRef: github_connector
          repoName: <+trigger.repo>
          events:
            - PullRequest
            - Push
          actions:
            - open
            - synchronize
            - closed
          payloadConditions:
            - key: sourceBranch
              operator: regex
              value: |
                ^(main|release/n\+1|release/n\+2|hotfix/CORENGC-[0-9]+|feature/CORENGC-[0-9]+)$
```

#### 2. Environment Detection Logic

```yaml
# Harness Pipeline Stage: Environment Detection
stages:
  - stage:
      name: Detect Environment
      identifier: detect_environment
      type: Custom
      spec:
        execution:
          steps:
            - step:
                type: ShellScript
                name: Detect Target Environment
                identifier: detect_env
                spec:
                  shell: Bash
                  source:
                    type: Inline
                    spec:
                      script: |
                        BRANCH="<+trigger.sourceBranch>"
                        
                        # Extract environment from branch
                        if [[ "$BRANCH" == "main" ]]; then
                          ENV_LAYER="BLD"
                          ENV_TYPE="stable"
                          ENV_INSTANCE="01"
                          TARGET_ENV="BLD-stable-01"
                        elif [[ "$BRANCH" =~ ^release/n\+1 ]]; then
                          ENV_LAYER="BLD"
                          ENV_TYPE="beta"
                          ENV_INSTANCE="01"
                          TARGET_ENV="BLD-beta-01"
                        elif [[ "$BRANCH" =~ ^release/n\+2 ]]; then
                          ENV_LAYER="BLD"
                          ENV_TYPE="alpha"
                          ENV_INSTANCE="01"
                          TARGET_ENV="BLD-alpha-01"
                        elif [[ "$BRANCH" =~ ^hotfix/CORENGC-[0-9]+ ]]; then
                          ENV_LAYER="BLD"
                          ENV_TYPE="stable"
                          ENV_INSTANCE="01"
                          TARGET_ENV="BLD-stable-01"
                        elif [[ "$BRANCH" =~ ^feature/CORENGC-[0-9]+ ]]; then
                          # Check base branch from PR
                          BASE_BRANCH="<+trigger.targetBranch>"
                          if [[ "$BASE_BRANCH" == "main" ]]; then
                            ENV_LAYER="BLD"
                            ENV_TYPE="stable"
                            TARGET_ENV="BLD-stable-01"
                          elif [[ "$BASE_BRANCH" =~ release/n\+1 ]]; then
                            ENV_LAYER="BLD"
                            ENV_TYPE="beta"
                            TARGET_ENV="BLD-beta-01"
                          fi
                        fi
                        
                        echo "Target Environment: $TARGET_ENV"
                        echo "Environment Layer: $ENV_LAYER"
                        echo "Release Type: $ENV_TYPE"
                        echo "Instance: $ENV_INSTANCE"
                        
                        # Set Harness variables
                        export TARGET_ENV
                        export ENV_LAYER
                        export ENV_TYPE
                        export ENV_INSTANCE
                  environmentVariables:
                    - name: TARGET_ENV
                      type: String
                      value: <+execution.steps.detect_env.output.outputVariables.TARGET_ENV>
                    - name: ENV_LAYER
                      type: String
                      value: <+execution.steps.detect_env.output.outputVariables.ENV_LAYER>
                    - name: ENV_TYPE
                      type: String
                      value: <+execution.steps.detect_env.output.outputVariables.ENV_TYPE>
```

#### 3. Dynamic Environment Creation

```yaml
# Harness Pipeline Stage: Create Environment if Needed
stages:
  - stage:
      name: Ensure Environment Exists
      identifier: ensure_environment
      type: Custom
      spec:
        execution:
          steps:
            - step:
                type: ShellScript
                name: Check and Create Environment
                identifier: create_env
                spec:
                  shell: Bash
                  source:
                    type: Inline
                    spec:
                      script: |
                        TARGET_ENV="<+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>"
                        ENV_LAYER="<+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.ENV_LAYER>"
                        ENV_TYPE="<+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.ENV_TYPE>"
                        
                        # Check if environment exists in Harness
                        ENV_EXISTS=$(harness-cli environment get "$TARGET_ENV" 2>/dev/null || echo "false")
                        
                        if [[ "$ENV_EXISTS" == "false" ]]; then
                          echo "Creating environment: $TARGET_ENV"
                          
                          # Create environment in Harness
                          harness-cli environment create \
                            --name "$TARGET_ENV" \
                            --type "Production" \
                            --tags "layer:$ENV_LAYER,type:$ENV_TYPE,instance:01"
                          
                          # Create infrastructure definition
                          harness-cli infrastructure create \
                            --env "$TARGET_ENV" \
                            --name "${TARGET_ENV}-infra" \
                            --type "Kubernetes" \
                            --yaml "infrastructure-definitions/${ENV_TYPE}.yaml"
                          
                          echo "Environment $TARGET_ENV created successfully"
                        else
                          echo "Environment $TARGET_ENV already exists"
                        fi
```

#### 4. Deployment Stage with Environment Promotion

```yaml
# Harness Pipeline Stage: Deploy and Promote
stages:
  - stage:
      name: Deploy to BLD
      identifier: deploy_bld
      type: Deployment
      spec:
        environment:
          name: <+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>
          type: Production
        service:
          name: <+service.name>
        infrastructure:
          name: <+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>-infra
        execution:
          steps:
            - step:
                type: K8sRollingDeploy
                name: Deploy to BLD
                identifier: deploy
                spec:
                  skipDryRun: false
            - step:
                type: Http
                name: Run Smoke Tests
                identifier: smoke_tests
                spec:
                  url: https://<+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>.example.com/health
                  method: GET
                  assertion: <+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>_health_check
                  
  - stage:
      name: Promote to INT
      identifier: promote_int
      type: Deployment
      dependsOn:
        - deploy_bld
      when:
        condition: <+pipeline.stages.deploy_bld.status> == "SUCCESS"
      spec:
        environment:
          name: <+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.ENV_LAYER>-<+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.ENV_TYPE>-02
          type: Production
        # ... deployment steps
```

## Multi-Service Release Orchestration

### Coordinated Release Across Services

#### 1. Release Plan Definition

```yaml
# Harness Release Plan: coordinated-release-v1.3.0.yml
releasePlan:
  name: Coordinated Release v1.3.0
  identifier: coordinated_release_v1_3_0
  version: v1.3.0
  targetBranch: release/n+1
  
  services:
    - name: user-service
      repository: github.com/org/user-service
      branch: release/n+1
      deploymentOrder: 1
      dependencies: []
      
    - name: payment-service
      repository: github.com/org/payment-service
      branch: release/n+1
      deploymentOrder: 2
      dependencies:
        - user-service
        
    - name: order-service
      repository: github.com/org/order-service
      branch: release/n+1
      deploymentOrder: 3
      dependencies:
        - user-service
        - payment-service
        
    - name: api-gateway
      repository: github.com/org/api-gateway
      branch: release/n+1
      deploymentOrder: 4
      dependencies:
        - user-service
        - payment-service
        - order-service

  environments:
    - name: BLD-beta-01
      type: Build
      order: 1
      
    - name: INT-beta-01
      type: Integration
      order: 2
      promotionGates:
        - type: AutomatedTests
        - type: ManualApproval
          approvers:
            - qa-team
            
    - name: PRE-beta-01
      type: PreProduction
      order: 3
      promotionGates:
        - type: AutomatedTests
        - type: ManualApproval
          approvers:
            - release-manager
```

#### 2. Orchestrated Deployment Pipeline

```yaml
# Harness Pipeline: Coordinated Release Pipeline
pipeline:
  name: Coordinated Release Pipeline
  identifier: coordinated_release
  
  variables:
    - name: releaseVersion
      type: String
      value: v1.3.0
    - name: targetBranch
      type: String
      value: release/n+1
    - name: services
      type: Array
      value:
        - user-service
        - payment-service
        - order-service
        - api-gateway

  stages:
    # Stage 1: Deploy all services to BLD-beta-01
    - stage:
        name: Deploy to BLD-beta-01
        identifier: deploy_bld_beta
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  name: Deploy All Services to BLD
                  identifier: deploy_all_bld
                  spec:
                    shell: Bash
                    source:
                      type: Inline
                      spec:
                        script: |
                          SERVICES=("user-service" "payment-service" "order-service" "api-gateway")
                          
                          for service in "${SERVICES[@]}"; do
                            echo "Deploying $service to BLD-beta-01..."
                            
                            # Trigger deployment pipeline for each service
                            harness-cli pipeline execute \
                              --pipeline "deploy-${service}" \
                              --branch "release/n+1" \
                              --environment "BLD-beta-01" \
                              --wait
                            
                            # Wait for deployment to complete
                            if [ $? -eq 0 ]; then
                              echo "✅ $service deployed successfully"
                            else
                              echo "❌ $service deployment failed"
                              exit 1
                            fi
                          done
                          
    # Stage 2: Run Integration Tests
    - stage:
        name: Integration Tests
        identifier: integration_tests
        type: Custom
        dependsOn:
          - deploy_bld_beta
        spec:
          execution:
            steps:
              - step:
                  type: RunTests
                  name: Run Integration Tests
                  identifier: run_tests
                  spec:
                    testType: Integration
                    environment: BLD-beta-01
                    services:
                      - user-service
                      - payment-service
                      - order-service
                      - api-gateway
                      
    # Stage 3: Promote to INT-beta-01 (in dependency order)
    - stage:
        name: Promote to INT-beta-01
        identifier: promote_int_beta
        type: Custom
        dependsOn:
          - integration_tests
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  name: Promote Services in Order
                  identifier: promote_services
                  spec:
                    shell: Bash
                    source:
                      type: Inline
                      spec:
                        script: |
                          # Deploy in dependency order
                          harness-cli pipeline execute --pipeline "deploy-user-service" --environment "INT-beta-01" --wait
                          harness-cli pipeline execute --pipeline "deploy-payment-service" --environment "INT-beta-01" --wait
                          harness-cli pipeline execute --pipeline "deploy-order-service" --environment "INT-beta-01" --wait
                          harness-cli pipeline execute --pipeline "deploy-api-gateway" --environment "INT-beta-01" --wait
```

## Environment Creation Workflow

### Automatic Environment Creation

#### 1. Environment Template

```yaml
# Harness Environment Template: environment-template.yaml
environment:
  name: ${ENV_NAME}
  identifier: ${ENV_IDENTIFIER}
  type: ${ENV_TYPE}  # Production, PreProduction, etc.
  tags:
    layer: ${ENV_LAYER}  # BLD, INT, PRE, PRD
    releaseType: ${RELEASE_TYPE}  # stable, beta, alpha
    instance: ${INSTANCE}  # 01, 02, etc.
    
  infrastructureDefinitions:
    - name: ${ENV_NAME}-infra
      type: Kubernetes
      spec:
        connectorRef: kubernetes_connector
        namespace: ${ENV_NAME}
        releaseName: release-${ENV_NAME}
        
  overrides:
    manifests:
      - manifest:
          identifier: values
          type: Values
          spec:
            store:
              type: Harness
              spec:
                files:
                  - ${ENV_TYPE}-values.yaml
```

#### 2. Environment Creation Script

```yaml
# Harness Pipeline: Create Environment
pipeline:
  name: Create Environment
  identifier: create_environment
  
  variables:
    - name: branchName
      type: String
    - name: releaseType
      type: String
    - name: instance
      type: String
      value: "01"

  stages:
    - stage:
        name: Create Environment
        identifier: create_env
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  name: Parse Branch and Create Environment
                  identifier: parse_branch
                  spec:
                    shell: Bash
                    source:
                      type: Inline
                      spec:
                        script: |
                          BRANCH="<+pipeline.variables.branchName>"
                          
                          # Determine environment details from branch
                          if [[ "$BRANCH" == "main" ]]; then
                            ENV_LAYER="BLD"
                            RELEASE_TYPE="stable"
                            ENV_TYPE="Production"
                          elif [[ "$BRANCH" =~ ^release/n\+1 ]]; then
                            ENV_LAYER="BLD"
                            RELEASE_TYPE="beta"
                            ENV_TYPE="PreProduction"
                          elif [[ "$BRANCH" =~ ^release/n\+2 ]]; then
                            ENV_LAYER="BLD"
                            RELEASE_TYPE="alpha"
                            ENV_TYPE="PreProduction"
                          fi
                          
                          INSTANCE="<+pipeline.variables.instance>"
                          ENV_NAME="${ENV_LAYER}-${RELEASE_TYPE}-${INSTANCE}"
                          ENV_IDENTIFIER=$(echo "$ENV_NAME" | tr '[:upper:]' '[:lower:]' | tr '-' '_')
                          
                          echo "Creating environment: $ENV_NAME"
                          echo "Environment Type: $ENV_TYPE"
                          echo "Release Type: $RELEASE_TYPE"
                          echo "Layer: $ENV_LAYER"
                          
                          # Create environment using Harness API/CLI
                          harness-cli environment create \
                            --name "$ENV_NAME" \
                            --identifier "$ENV_IDENTIFIER" \
                            --type "$ENV_TYPE" \
                            --tags "layer:$ENV_LAYER,releaseType:$RELEASE_TYPE,instance:$INSTANCE" \
                            --yaml "environment-templates/${RELEASE_TYPE}-environment.yaml"
                          
                          # Create infrastructure definition
                          harness-cli infrastructure create \
                            --env "$ENV_NAME" \
                            --name "${ENV_NAME}-infra" \
                            --type "Kubernetes" \
                            --yaml "infrastructure-templates/${RELEASE_TYPE}-infra.yaml"
                          
                          echo "✅ Environment $ENV_NAME created successfully"
```

## Release Orchestration Benefits

### 1. Automated Environment Selection

**Before**: Manual environment selection, prone to errors
**After**: Automatic environment detection based on branch pattern

```yaml
# Branch: release/n+1
# Automatically deploys to: BLD-beta-01 → INT-beta-01 → PRE-beta-01
```

### 2. Coordinated Multi-Service Releases

**Before**: Manual coordination, deployment order errors
**After**: Automated dependency-based deployment

```yaml
# Services deploy in correct order:
# 1. user-service (no dependencies)
# 2. payment-service (depends on user-service)
# 3. order-service (depends on payment-service, user-service)
# 4. api-gateway (depends on all)
```

### 3. Dynamic Environment Creation

**Before**: Manual environment setup, inconsistent configurations
**After**: Automatic environment creation with consistent templates

```yaml
# New branch: release/n+1
# Automatically creates: BLD-beta-02, INT-beta-02 (if needed)
```

### 4. Environment Promotion Gates

**Before**: Manual promotion, inconsistent approval process
**After**: Automated gates with clear approval requirements

```yaml
# BLD → INT: Automated (tests pass)
# INT → PRE: Manual approval (QA team)
# PRE → PRD: Manual approval (Release manager)
```

## Harness-Specific Configurations

### 1. Harness Service Definition

```yaml
# Harness Service: service-definition.yaml
service:
  name: user-service
  identifier: user_service
  tags:
    - service:user-service
    - team:platform
  
  gitOpsEnabled: true
  gitOps:
    repo: github.com/org/user-service
    branch: <+trigger.sourceBranch>
    
  manifests:
    - manifest:
        identifier: k8s-manifest
        type: K8sManifest
        spec:
          store:
            type: Git
            spec:
              connectorRef: github_connector
              gitFetchType: Branch
              paths:
                - k8s/manifests/
              repoName: user-service
              branch: <+trigger.sourceBranch>
```

### 2. Harness Environment Definition

```yaml
# Harness Environment: BLD-beta-01.yaml
environment:
  name: BLD-beta-01
  identifier: bld_beta_01
  type: PreProduction
  tags:
    layer: BLD
    releaseType: beta
    instance: "01"
    
  infrastructureDefinitions:
    - name: BLD-beta-01-infra
      identifier: bld_beta_01_infra
      type: KubernetesDirect
      spec:
        connectorRef: kubernetes_connector
        namespace: bld-beta-01
        releaseName: release-bld-beta-01
```

### 3. Harness Pipeline with Branch Detection

```yaml
# Harness Pipeline: service-deployment-pipeline.yaml
pipeline:
  name: Service Deployment Pipeline
  identifier: service_deployment
  
  variables:
    - name: targetEnvironment
      type: String
      value: <+pipeline.stages.detect_environment.spec.execution.steps.detect_env.output.outputVariables.TARGET_ENV>
      
  stages:
    - stage:
        name: Detect Environment
        identifier: detect_environment
        # ... (as shown above)
        
    - stage:
        name: Build and Deploy
        identifier: build_deploy
        type: Deployment
        dependsOn:
          - detect_environment
        spec:
          environment:
            name: <+pipeline.variables.targetEnvironment>
          service:
            name: <+service.name>
          execution:
            steps:
              - step:
                  type: BuildAndPushGCR
                  name: Build Docker Image
                  identifier: build_image
                  spec:
                    connectorRef: gcp_connector
                    image: gcr.io/project/<+service.name>
                    tags:
                      - <+pipeline.variables.targetEnvironment>-<+codebase.commitSha>
                      
              - step:
                  type: K8sRollingDeploy
                  name: Deploy
                  identifier: deploy
                  spec:
                    skipDryRun: false
```

## Best Practices

### 1. Environment Naming Consistency

- ✅ Use consistent naming: `{LAYER}-{RELEASE_TYPE}-{INSTANCE}`
- ✅ Store naming convention in Harness templates
- ✅ Validate names in pipeline

### 2. Branch Pattern Validation

- ✅ Validate branch names match expected patterns
- ✅ Fail fast if branch doesn't match pattern
- ✅ Provide clear error messages

### 3. Dependency Management

- ✅ Define service dependencies in release plan
- ✅ Enforce deployment order
- ✅ Wait for dependencies before deploying

### 4. Environment Lifecycle

- ✅ Auto-create environments on first use
- ✅ Auto-cleanup temporary environments (feature branches)
- ✅ Archive old environments after release promotion

### 5. Monitoring and Observability

- ✅ Track deployments per environment
- ✅ Monitor promotion times
- ✅ Alert on deployment failures
- ✅ Track service dependencies

## Example: Complete Release Orchestration Flow

### Scenario: Coordinated Release v1.3.0

```yaml
# 1. Release Plan Created
releasePlan:
  version: v1.3.0
  targetBranch: release/n+1
  services: [user-service, payment-service, order-service, api-gateway]

# 2. Environments Created (if needed)
environments:
  - BLD-beta-01
  - INT-beta-01
  - PRE-beta-01

# 3. Services Deployed to BLD-beta-01 (in order)
deployments:
  - user-service → BLD-beta-01 ✅
  - payment-service → BLD-beta-01 ✅ (waited for user-service)
  - order-service → BLD-beta-01 ✅ (waited for payment-service, user-service)
  - api-gateway → BLD-beta-01 ✅ (waited for all)

# 4. Integration Tests Run
tests:
  - Integration tests on BLD-beta-01 ✅

# 5. Promote to INT-beta-01 (in order)
promotions:
  - user-service → INT-beta-01 ✅
  - payment-service → INT-beta-01 ✅
  - order-service → INT-beta-01 ✅
  - api-gateway → INT-beta-01 ✅

# 6. QA Approval
approvals:
  - INT-beta-01 → PRE-beta-01: QA approved ✅

# 7. Promote to PRE-beta-01
promotions:
  - All services → PRE-beta-01 ✅

# 8. Release Manager Approval
approvals:
  - PRE-beta-01 → PRD-stable-01: Release manager approved ✅

# 9. Merge release/n+1 → main
git:
  - release/n+1 → main ✅

# 10. Deploy to Production
deployments:
  - All services → PRD-stable-01 ✅
```

## Conclusion

The branching strategy enables:

1. ✅ **Automated Environment Detection**: Branch patterns determine target environments
2. ✅ **Dynamic Environment Creation**: New environments created based on branch patterns
3. ✅ **Coordinated Releases**: Multi-service releases orchestrated automatically
4. ✅ **Dependency Management**: Services deploy in correct order
5. ✅ **Promotion Gates**: Automated gates with manual approvals where needed
6. ✅ **Consistency**: Standardized environment naming and configuration

This integration with Harness (or similar tools) transforms manual release processes into automated, reliable, and scalable workflows.

